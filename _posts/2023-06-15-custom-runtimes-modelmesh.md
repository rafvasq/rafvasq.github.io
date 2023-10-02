---
layout: post
title:  "Creating a custom serving runtime in KServe ModelMesh
"
date:   2023-06-15
categories: modelmesh
---


**_The following tutorial was originally posted on [IBM Developer](https://developer.ibm.com/tutorials/awb-creating-custom-runtimes-in-modelmesh/)._**

---

[ModelMesh](https://github.com/kserve/modelmesh) is a mature, general-purpose model serving management and routing layer. Optimized for high volume, high density, and frequently changing model use cases, ModelMesh intelligently loads and unloads models to and from memory to strike a balance between responsiveness and compute.

IBM used ModelMesh in production for several years before it was contributed to the open source community as part of [KServe](https://kserve.github.io/website). ModelMesh served as the backbone for most Watson services, such as Watson NLU and Watson Assistant, and more recently it underpins the upcoming enterprise-ready AI and data platform <a href="https://www.ibm.com/watsonx" target="_blank" rel="noopener noreferrer">**watsonx**</a>.

You can learn more about ModelMesh's features and install [ModelMesh Serving](https://github.com/kserve/modelmesh-serving), the controller for managing ModelMesh clusters, by following our <a href="https://developer.ibm.com/tutorials/awb-get-started-with-kserve-modelmesh-for-multi-model-serving/" target="_blank" rel="noopener noreferrer">getting started with KServe ModelMesh tutorial</a>. You'll even be guided to deploy and inference your first model!

[ModelMesh Serving](https://github.com/kserve/modelmesh-serving) provides many model servers by default, such as:

- [Triton Inference Server](https://developer.nvidia.com/nvidia-triton-inference-server), NVIDIA's server for frameworks like TensorFlow, PyTorch, TensorRT, or ONNX.
- [MLServer](https://mlserver.readthedocs.io/en/stable/), Seldon's Python-based server for frameworks like SKLearn, XGBoost, or LightGBM.
- [OpenVINO Model Server](https://docs.openvino.ai/2023.0/ovms_what_is_openvino_model_server.html), Intel's server for frameworks such as Intel OpenVINO or ONNX.
- [TorchServe](https://pytorch.org/serve/), support for PyTorch models that include eager mode.

However, these model servers might not meet all of your specific requirements. Your model might have custom functionality, it might have custom logic for inferencing, or the framework your model needs might not be supported yet (but feel free to [request it](https://github.com/kserve/modelmesh-serving/issues/new?assignees=&labels=&projects=&template=feature_request.md&title=)!).

In this tutorial, you'll learn how to serve your custom models by using ModelMesh Serving.

## Serving runtimes

The namespace-scoped `ServingRuntime` (and its cluster-scoped counterpart `ClusterServingRuntime`) defines the templates for pods that can serve one or more particular model formats. It includes key information such as the container image of the runtime and a list of the supported model formats while other configuration settings for the runtime can be passed through environment variables in the specification.

The `ServingRuntime` CRDs allow for flexibility and extensibility, which can help you to define or customize reusable runtimes without touching any of the ModelMesh controller code or other resources in the controller namespace. This means you can easily build a custom runtime to support your desired framework.

Custom serving runtimes are created by building a new container image with support for the desired framework and then creating a `ServingRuntime` resource that uses that image. This is especially easy if the desired runtime's framework uses Python bindings. For that scenario, there's a simplified process that uses MLServer's extension point for adding additional frameworks. MLServer provides the serving interface, you provide the framework, and ModelMesh Serving provides the glue to integrate it as a `ServingRuntime`.

## Build a Python-based custom serving runtime

At a high level, the steps required to build a custom runtime are:

1. Implement a class that inherits from [MLServer's `MLModel` class](https://github.com/SeldonIO/MLServer/blob/master/mlserver/model.py).

2. Package the model class and dependencies into a container image.

3. Create the new `ServingRuntime` resource using that image.

### Implement the MLModel class

MLServer can be extended by adding an implementation of the `MLModel` class. The two main functions are `load()` and `predict()`. The following code is a template implementation of an `MLModel` class in MLServer that includes a recommended structure along with *TODOs* where runtime-specific changes might need to be made. Another example implemention of this class can be found in the [MLServer documentation](https://mlserver.readthedocs.io/en/stable/examples/custom/README.html#training).

```python
from typing import List
from mlserver import MLModel, types
from mlserver.utils import get_model_uri

class CustomMLModel(MLModel):
  async def load(self) -> bool:
	model_uri = await get_model_uri(self._settings)
    self._load_model_from_file(model_uri)
    self.ready = True
    return self.ready

  async def predict(self, payload: types.InferenceRequest) -> types.InferenceResponse:

    payload = self._check_request(payload)

    return types.InferenceResponse(
      model_name=self.name,
      model_version=self.version,
      outputs=self._predict_outputs(payload),
    )

    def _load_model_from_file(self, file_uri):
        # TODO: load model from file and instantiate class data
        return

    def _check_request(self, payload: types.InferenceRequest) -> types.InferenceRequest:
        # TODO: validate request: number of inputs, input tensor names/types, etc.
        return payload

    def _predict_outputs(self, payload: types.InferenceRequest) -> List[types.ResponseOutput]:
        inputs = payload.inputs
		# TODO: transform inputs into internal data structures
        # TODO: send data through the model's prediction logic
		outputs = []
		return outputs
```

### Create the runtime image

Now that we have our model class implemented, we need to package its dependencies, including MLServer, into an image that is supported as a `ServingRuntime` resource. There are a variety of ways to do this, and [MLServer provides helpers](https://mlserver.readthedocs.io/en/latest/examples/custom/README.html#building-a-custom-image) to build it for you using the `mlserver build` command. Alternatively, you can build off of the set of directives included in the `Dockerfile` snippet in the following code block. (You can learn more about Dockerfiles in this [Docker tutorial](https://docker-curriculum.com/#dockerfile).)

```Dockerfile
# TODO: choose appropriate base image, install Python, MLServer, and
# dependencies of your MLModel implementation
FROM python:3.8-slim-buster
RUN pip install mlserver
# ...

# The custom `MLModel` implementation should be on the Python search path
# instead of relying on the working directory of the image. If using a
# single-file module, this can be accomplished with:
COPY --chown=${USER} ./custom_model.py /opt/custom_model.py
ENV PYTHONPATH=/opt/

# The environment variables here are for compatibility with ModelMesh Serving.
# These can also be set in the ServingRuntime, but this is recommended for
# consistency when building and testing
ENV MLSERVER_MODELS_DIR=/models/_mlserver_models \
    MLSERVER_GRPC_PORT=8001 \
    MLSERVER_HTTP_PORT=8002 \
    MLSERVER_LOAD_MODELS_AT_STARTUP=false \
    MLSERVER_MODEL_NAME=dummy-model

# With this setting, the implementation field is not required in the model
# settings which eases integration by allowing the built-in adapter to generate
# a basic model settings file
ENV MLSERVER_MODEL_IMPLEMENTATION=custom_model.CustomMLModel

CMD ["mlserver", "start", "${MLSERVER_MODELS_DIR}"]
```

### Create the ServingRuntime resource

Now you can make a new `ServingRuntime` resource using the YAML template in the following code block and point it to the image you just created. In this YAML code:

- `{{CUSTOM-RUNTIME-NAME}}` is the name you want to give your runtime (for example, `my-custom-runtime-0.x`).

- `{{MODEL-FORMAT-NAMES}}` is a list of model formats that this runtime will support. Behind the scenes, this is what ModelMesh will look for when deploying a model of that format and finding a suitable runtime for it.

- `{{CUSTOM-IMAGE-NAME}}` refers to the image you created in the previous step.


```yaml
apiVersion: serving.kserve.io/v1alpha1
kind: ServingRuntime
metadata:
  name: {{CUSTOM-RUNTIME-NAME}}
spec:
  supportedModelFormats:
    - name: {{MODEL-FORMAT-NAMES}}
      version: "1"
      autoSelect: true
  multiModel: true
  grpcDataEndpoint: port:8001
  grpcEndpoint: port:8085
  containers:
    - name: mlserver
      image: {{CUSTOM-IMAGE-NAME}}
      env:
        - name: MLSERVER_MODELS_DIR
          value: "/models/_mlserver_models/"
        - name: MLSERVER_GRPC_PORT
          value: "8001"
        - name: MLSERVER_HTTP_PORT
          value: "8002"
        - name: MLSERVER_LOAD_MODELS_AT_STARTUP
          value: "false"
        - name: MLSERVER_MODEL_NAME
          value: dummy-model
        - name: MLSERVER_HOST
          value: "127.0.0.1"
        - name: MLSERVER_GRPC_MAX_MESSAGE_LENGTH
          value: "-1"
      resources:
        requests:
          cpu: 500m
          memory: 1Gi
        limits:
          cpu: "5"
          memory: 1Gi
  builtInAdapter:
    serverType: mlserver
    runtimeManagementPort: 8001
    memBufferBytes: 134217728
    modelLoadingTimeoutMillis: 90000
```

And that's it! Create the `ServingRuntime` resource using the `kubectl apply` command, and you'll see your new custom runtime in your ModelMesh deployment. In the following example output, the custom runtime's name is `custom-runtime-0.x` and it supports the model format `custom_model`.

```shell
kubectl get servingruntimes
```
```
NAME                 DISABLED   MODELTYPE     CONTAINERS   AGE
custom-runtime-0.x              custom_model  mlserver     5m
mlserver-1.x                    sklearn       mlserver     32m
ovms-1.x                        openvino_ir   ovms         32m
torchserve-0.x                  pytorch-mar   torchserve   32m
triton-2.x                      keras         triton       31m
```

### Deploy your model

To deploy a model using your newly created runtime, you'll need to create an `InferenceService` resource to serve the model. This resource  is the main interface that KServe and ModelMesh use for managing models, representing the model's logical endpoint for serving inferences.

```yaml
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: sentiment-analyzer
  annotations:
    serving.kserve.io/deploymentMode: ModelMesh
spec:
  predictor:
    model:
      modelFormat:
        name: custom-model
      runtime: custom-runtime-0.x # OPTIONAL
      storage:
        key: localMinIO
        path: sentiment-model/models.py
```

The `InferenceService` in the previous code block names the model `sentiment-analyzer` and declares its model format `custom-model`, which is the same format as the example custom runtime created earlier. An optional field `runtime` is passed as well, explictly telling ModelMesh to use the `custom-runtime-0.x` serving runtime to deploy this model. Lastly, the `storage` field points to where the model resides, in this case the `localMinIO` instance that deploys using ModelMesh Serving's [quickstart guide](https://github.com/kserve/modelmesh-serving/blob/main/docs/quickstart.md).

After creating the `InferenceService`, you'll be able to watch it become available as shown in the following sample output:

```shell
kubectl get isvc
```
```
NAME                URL                                               READY   ...
sentiment-analyzer   grpc://modelmesh-serving.modelmesh-serving:8033   True    ...
```

## Summary and next steps

Now that you can build your own custom runtime to deploy any model you like, there's nothing stopping you from taking advantage of ModelMesh's effectiveness and reliability to scale as needed.

If you want an enterprise-grade platform for your AI workloads built on top of open source software like ModelMesh, be sure to check out <a href="https://www.ibm.com/watsonx" target="_blank" rel="noopener noreferrer">**watsonx**</a>. Explore more <a href="https://developer.ibm.com/components/watsonx" target="_blank" rel="noopener noreferrer">articles and tutorials about watsonx</a> on IBM Developer.
