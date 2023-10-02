---
layout: post
title:  "Get started with KServe ModelMesh for multi-model serving
"
date:   2023-06-15
categories: modelmesh
---

_The following blog was originally posted and revised on [IBM Developer](https://developer.ibm.com/tutorials/awb-get-started-with-kserve-modelmesh-for-multi-model-serving/)._


[ModelMesh](https://github.com/kserve/modelmesh) is a mature, general-purpose model serving management and routing layer. Optimized for high volume, high density, and frequently changing model use cases, ModelMesh intelligently loads and unloads models to and from memory to strike a balance between responsiveness and compute.

You can read more about ModelMesh features in [this blog](https://developer.ibm.com/blogs/kserve-and-watson-modelmesh-extreme-scale-model-inferencing-for-trusted-ai/) but here are some of its features at a glance:

- **Cache management**
  - Pods are managed as a distributed least recently used (LRU) cache.
  - Copies of models are loaded and unloaded based on usage recency and current request volumes.
- **Intelligent placement and loading**
  - Model placement is balanced by both the cache age across the pods and the request load.
  - Queues are used to handle concurrent model loads and minimize impact to runtime traffic.
- **Resiliency**
  - Failed model loads are automatically retried in different pods.
- **Operational simplicity**
  - Rolling model updates are handled automatically and seamlessly.

IBM used ModelMesh in production for several years before it was contributed to the open source community as part of [KServe](https://kserve.github.io/website). ModelMesh has served as the backbone for most Watson services, such as Watson NLU and watsonx Assistant, and more recently it underpins the upcoming enterprise-ready AI and data platform <a href="https://www.ibm.com/watsonx" target="_blank" rel="noopener noreferrer">**watsonx**</a>.

In this tutorial, you'll be guided to:

1. Install [ModelMesh Serving](https://github.com/kserve/modelmesh-serving), the controller for managing ModelMesh clusters.
2. Deploy a model using the `InferenceService` resource and check its status.
3. Perform an inference on the deployed model using gRPC and REST.

## Prerequisites

The following prerequisites are required to install ModelMesh Serving:

- A Kubernetes cluster (v1.16+) with administrative privileges.
- [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl) and [kustomize](https://kubectl.docs.kubernetes.io/installation/kustomize/) (v3.2.0+).
- At least 4 vCPUs and 8 GB memory. If you'd like information about resources, check out the details about each [deployed component](https://github.com/kserve/modelmesh-serving/tree/main/docs/install#deployed-components) of ModelMesh Serving.

## Step 1. Install ModelMesh Serving

Assuming the latest release is `v0.10`, start by cloning the branch:

```shell
RELEASE=release-0.10
git clone -b $RELEASE --depth 1 --single-branch https://github.com/kserve/modelmesh-serving.git
cd modelmesh-serving
```

From within our cloned repository, we can run the installation script. Create a namespace called `modelmesh-serving` to deploy ModelMesh to. You'll pass the namespace as a parameter for the script. You'll also include the flag `--namespace-scope-mode`, which means that the ModelMesh Serving instance and its components will exist only within a single namespace. (The alternative (and default) is cluster-scoped.) Lastly, the `--quickstart` flag will make sure `etcd` and `MinIO` instances with sample models are included in the installation too. These sample models are intended to be used for development and experimentation and not for production. *If you want to dive deeper, check out [the installation documentation](https://github.com/kserve/modelmesh-serving/blob/main/docs/install/install-script.md "the installation documentation").*

The script should only take a few minutes to run and will output the message `Successfully installed ModelMesh Serving!` when completed succesfully.

Let's verify the installation.

First, make sure that the controller, `MinIO`, and the `etcd` pods are running:

```shell
kubectl get pods
```
```
NAME                                        READY   STATUS    RESTARTS   AGE
pod/etcd                                    1/1     Running   0          5m
pod/minio                                   1/1     Running   0          5m
pod/modelmesh-controller-547bfb64dc-mrgrq   1/1     Running   0          5m
```

Now, let's confirm that the `ServingRuntime` resources are available:

```shell
kubectl get servingruntimes
```
```
NAME           DISABLED   MODELTYPE    CONTAINERS   AGE
mlserver-1.x              sklearn      mlserver     5m
ovms-1.x                  openvino_ir  ovms         5m
torchserve-0.x            pytorch-mar  torchserve   5m
triton-2.x                tensorflow   triton       5m
```

A `ServingRuntime` defines the templates for pods that can serve one or more particular model formats pods for each are automatically provisioned depending on the framework of the model deployed. ModelMesh Serving currently includes several runtimes by default:

| ServingRuntime | [Supported Frameworks](https://github.com/kserve/modelmesh-serving/tree/main/docs/model-formats#supported-model-formats "Supported Frameworks")                |
| -------------- | ----------------------------------- |
| mlserver-1.x   | sklearn, xgboost, lightgbm          |
| ovms-1.x       | openvino_ir, onnx                   |
| torchserve-0.x | pytorch-mar                         |
| triton-2.x     | tensorflow, pytorch, onnx, tensorrt |

You can learn more about serving runtimes [here](https://github.com/kserve/modelmesh-serving/tree/main/docs/runtimes), or if these model servers don't meet all of your specific requirements, you can learn how to build your own custom serving runtime in this tutorial, <a href="https://developer.ibm.com/tutorials/awb-creating-custom-runtimes-in-modelmesh/" target="_blank" rel="noopener noreferrer">"Creating a custom serving runtime in KServe ModelMesh"</a>.

## Step 2. Deploy a model

With ModelMesh Serving now installed, you can deploy a model using the KServe `InferenceService` custom resource definition. It's the main interface that KServe and ModelMesh use for managing models, representing the model's logical endpoint for serving inferences. The ModelMesh controller will only handle those that include the `serving.kserve.io/deploymentMode: ModelMesh` annotation.

As mentioned earlier, deploying ModelMesh Serving using the `--quickstart` flag includes [a set of sample models](https://github.com/kserve/modelmesh-serving/blob/main/docs/example-models.md) to get started with. Below, we'll deploy the sample `SKLearn MNIST` model served from the local `MinIO` container:

```shell
kubectl apply -f - <<EOF
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: example-sklearn-isvc
  annotations:
    serving.kserve.io/deploymentMode: ModelMesh
spec:
  predictor:
    model:
      modelFormat:
        name: sklearn
      storage:
        key: localMinIO
        path: sklearn/mnist-svm.joblib
EOF
```

The above YAML uses the `InferenceService` predictor [`storageSpec`](https://github.com/kserve/kserve/tree/master/docs/samples/storage/storageSpec) where the `key` is the credential key for the destination storage in the common secret and the `path` is the model path inside the bucket. You could also use [`storageUri`](https://github.com/kserve/kserve/tree/master/docs/samples/storage/uri "`storageUri`") instead:

```shell
kubectl apply -f - <<EOF
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: example-sklearn-isvc
  annotations:
    serving.kserve.io/deploymentMode: ModelMesh
    serving.kserve.io/secretKey: localMinIO
spec:
  predictor:
    model:
      modelFormat:
        name: sklearn
      storageUri: s3://modelmesh-example-models/sklearn/mnist-svm.joblib
EOF
```

Either way you do it, after creating this `InferenceService`, you'll likely see that it's not yet ready:


```shell
kubectl get isvc
```
```
NAME                    URL   READY   PREV   LATEST   PREVROLLEDOUTREVISION   LATESTREADYREVISION   AGE
example-sklearn-isvc          False                                                                 3s
```

If you do, it's probably because the `ServingRuntime` pods that will host the `SKLearn` model are still spinning up. You can check your pods to confirm this and eventually, you'll see them in the  `Running` state:

```shell
kubectl get pods
```
```
...
modelmesh-serving-mlserver-1.x-7db675f677-twrwd   3/3     Running   0          2m
modelmesh-serving-mlserver-1.x-7db675f677-xvd8q   3/3     Running   0          2m
```

Then, checking on the `InferenceService` again, it is now ready with a provided URL:

```shell
kubectl get isvc
```
```
NAME                    URL                                               READY   PREV   LATEST   PREVROLLEDOUTREVISION   LATESTREADYREVISION   AGE
example-sklearn-isvc    grpc://modelmesh-serving.modelmesh-serving:8033   True                                                                  97s
```

You can describe the `InferenceService` to get descriptive status information too. If you checked it before it was ready, you will see useful information for debugging such as a `Waiting for runtime Pod to become available` message, but not in this case:

```shell
kubectl describe isvc example-sklearn-isvc
```
```
Name:         example-sklearn-isvc
...
Status:
  Components:
    Predictor:
      Grpc URL:  grpc://modelmesh-serving.modelmesh-serving:8033
      Rest URL:  http://modelmesh-serving.modelmesh-serving:8008
      URL:       grpc://modelmesh-serving.modelmesh-serving:8033
  Conditions:
    Last Transition Time:  2022-07-18T18:01:54Z
    Status:                True
    Type:                  PredictorReady
    Last Transition Time:  2022-07-18T18:01:54Z
    Status:                True
    Type:                  Ready
  Model Status:
    Copies:
      Failed Copies:  0
      Total Copies:   2
    States:
      Active Model State:  Loaded
      Target Model State:
    Transition Status:     UpToDate
  URL:                     grpc://modelmesh-serving.modelmesh-serving:8033
...
```

More details on the `InferenceService` CRD and its status information can be found [the ModelMesh Serving docs](https://github.com/kserve/modelmesh-serving/blob/main/docs/predictors/inferenceservice-cr.md).

## Step 3. Perform an inference request

Now that a model is loaded and available, you can perform inferences! Currently, only gRPC inference requests are supported by ModelMesh, but REST support is enabled via a [REST proxy](https://github.com/kserve/rest-proxy) container. By default, ModelMesh Serving uses a
[headless Service](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services) since [load balancing gRPC requests](https://kubernetes.io/blog/2018/11/07/grpc-load-balancing-on-kubernetes-without-tears/) requires special attention.

### Inferencing using gRPC request

To test out gRPC inference requests, you can port-forward the headless service in a separate terminal window:

```shell
kubectl port-forward --address 0.0.0.0 service/modelmesh-serving 8033 -n modelmesh-serving
```

Using [grpcurl](https://github.com/fullstorydev/grpcurl) and the gRPC client generated from the KServe [grpc_predict_v2.proto](https://github.com/kserve/kserve/blob/master/docs/predict-api/v2/grpc_predict_v2.proto) file, we can test an inference on `localhost:8033`. Make sure to run the following command from the root directory of the cloned repository and set `MODEL_NAME` to the name of the deployed `InferenceService`:

```shell
MODEL_NAME=example-sklearn-isvc
grpcurl \
  -plaintext \
  -proto fvt/proto/kfs_inference_v2.proto \
  -d '{ "model_name": "'"${MODEL_NAME}"'", "inputs": [{ "name": "predict", "shape": [1, 64], "datatype": "FP32", "contents": { "fp32_contents": [0.0, 0.0, 1.0, 11.0, 14.0, 15.0, 3.0, 0.0, 0.0, 1.0, 13.0, 16.0, 12.0, 16.0, 8.0, 0.0, 0.0, 8.0, 16.0, 4.0, 6.0, 16.0, 5.0, 0.0, 0.0, 5.0, 15.0, 11.0, 13.0, 14.0, 0.0, 0.0, 0.0, 0.0, 2.0, 12.0, 16.0, 13.0, 0.0, 0.0, 0.0, 0.0, 0.0, 13.0, 16.0, 16.0, 6.0, 0.0, 0.0, 0.0, 0.0, 16.0, 16.0, 16.0, 7.0, 0.0, 0.0, 0.0, 0.0, 11.0, 13.0, 12.0, 1.0, 0.0] }}]}' \
  localhost:8033 \
  inference.GRPCInferenceService.ModelInfer
```

The response output will look like:

```json
{
  "modelName": "example-sklearn-isvc__isvc-3642375d03",
  "outputs": [
    {
      "name": "predict",
      "datatype": "INT64",
      "shape": ["1"],
      "contents": {
        "int64Contents": ["8"]
      }
    }
  ]
}
```

### Inferencing using a REST request

While the [REST proxy](https://github.com/kserve/rest-proxy) is currently in an alpha state, we can still use `curl` to test an inference using it as well.

First you'll need to port-forward a different port for REST:

```shell
kubectl port-forward --address 0.0.0.0 service/modelmesh-serving 8008 -n modelmesh-serving
```

Once again, make sure that `MODEL_NAME` is set to the name of your `InferenceService` when you run the following command:

```shell
MODEL_NAME=example-sklearn-isvc
curl -X POST -k http://localhost:8008/v2/models/${MODEL_NAME}/infer -d '{"inputs": [{ "name": "predict", "shape": [1, 64], "datatype": "FP32", "data": [0.0, 0.0, 1.0, 11.0, 14.0, 15.0, 3.0, 0.0, 0.0, 1.0, 13.0, 16.0, 12.0, 16.0, 8.0, 0.0, 0.0, 8.0, 16.0, 4.0, 6.0, 16.0, 5.0, 0.0, 0.0, 5.0, 15.0, 11.0, 13.0, 14.0, 0.0, 0.0, 0.0, 0.0, 2.0, 12.0, 16.0, 13.0, 0.0, 0.0, 0.0, 0.0, 0.0, 13.0, 16.0, 16.0, 6.0, 0.0, 0.0, 0.0, 0.0, 16.0, 16.0, 16.0, 7.0, 0.0, 0.0, 0.0, 0.0, 11.0, 13.0, 12.0, 1.0, 0.0]}]}'
```

The response output will look like:

```json
{
  "model_name": "example-sklearn-isvc__ksp-7702c1b55a",
  "outputs": [
    {
      "name": "predict",
      "datatype": "FP32",
      "shape": [1],
      "data": [8]
    }
  ]
}
```

You've made your first inference requests! As always, more detailed information about sending inference requests to your `InferenceService` can be found in the [ModelMesh Serving docs]](https://github.com/kserve/modelmesh-serving/blob/main/docs/predictors/run-inference.md).

## Summary and next steps

This tutorial is a great starting point toward taking advantage of ModelMesh's effectiveness and reliability to scale as needed. You learned about some of ModelMesh's features and core resources like the `ServingRuntime` and the `InferenceService`, all while deploying and inferencing your first model deployed on your own ModelMesh Serving instance.

You can dive deeper and learn how to create a custom serving runtime to serve any model you'd like in this tutorial, <a href="https://developer.ibm.com/tutorials/awb-creating-custom-runtimes-in-modelmesh/" target="_blank" rel="noopener noreferrer">"Creating a custom serving runtime in KServe ModelMesh"</a>.

If you want an enterprise-grade platform for your AI workloads built on top of open source software like ModelMesh, be sure to try <a href="https://www.ibm.com/watsonx?cm_sp=ibmdev-_-developer-_-trial" target="_blank" rel="noopener noreferrer">**watsonx**</a>. Explore more <a href="https://developer.ibm.com/components/watsonx" target="_blank" rel="noopener noreferrer">articles and tutorials about watsonx</a> on IBM Developer.

