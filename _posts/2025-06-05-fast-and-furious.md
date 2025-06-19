---
layout: post
title:  "Fast Inference and Furious Scaling with vLLM and KServe"
date:   2025-06-05
categories: vllm kserve
---

_Why serving large language models is hard — and how vLLM + KServe can help beginners get started._

The name "Fast Inference, Furious Scaling", obviously inspired by the movies, is not just catchy, but also captures the pace of this area of technology.  With new optimization techniques, tools emerging daily, and not enough "good first issues", it's a rapidly evolving landscape that often leaves beginners struggling to find their footing.

In this post, beginners who are new to the world of LLM inferencing and serving can learn about why it's a complicated thing to do and gain a clearer idea of how to get started with it using two open source tools: vLLM and KServe. Instead of bogging down in technical details, the focus is on the 'why' and 'how' to give a background context for those who want to participate, while including resources for further deep-diving linked along the way.

## What Does It Mean to "Serve" an LLM?

**Model serving** boils down to making a pre-trained model useable. When you try a cloud-based service like ChatGPT, models have been made available for you to send prompts (i.e. inference requests) and receive a response (i.e. output). Those models are being **served** for you to consume. Behind the scenes, the model has been made available through an API.

Sounds simple, right? Well, not quite.

Serving LLMs isn't just about wrapping them in an API. Just like any memory-intensive software program, large models bring performance trade-offs, infrastructure constraints, and cost considerations.

### Why It's Not So Simple

#### _Massive LLMs require massive resources._

[Hosting the 70-billion parameter version of Meta's Llama-3](https://abhinand05.medium.com/self-hosting-llama-3-1-70b-or-any-70b-llm-affordably-2bd323d72f8d) can require ~150GB of vRAM. OpenAI's GPT-4 is estimated to have [over 1.5 trillion parameters](https://the-decoder.com/gpt-4-architecture-datasets-costs-and-more-leaked/), just to give you an idea. Hosting these kinds of models require high-end GPUs (e.g. A100s or H100s), going beyond your average gaming rig or cloud instance. Running them efficiently and reliably requires specialized resources and infrastructure.

Those top-tier GPUs aren’t just expensive to buy, they’re expensive to operate. And in 2025, [they're scarce too](https://www.independent.co.uk/news/business/business-reporter/ai-gpu-data-centres-cooling-infrastructures-b2744764.html).

#### _It's their complexity, too._

Aside from their size, LLMs face technical hurdles under the hood.

- **[Tokenization](https://www.ibm.com/think/topics/tokenization)**: Each prompt they receive has to be processed or *tokenized*.
- **[Autoregressive generation](https://www.parlant.io/blog/what-is-autoregression/)**:  LLMs generate *one token at a time*. After predicting a token, that token becomes part of the input to predict the next one. In many cases, compute requirements scale quadratically with the length of a sequence: for instance, if the number of input tokens doubles, the model needs 4 times as much processing power to handle it.
- **[Context window](https://www.ibm.com/think/topics/context-window)**: An LLM's working memory. Every time the model predicts the next token in a sequence, it's computing the relationships between that token and **every token before it**. So, longer prompts and conversation result in slower inference.

#### _Scaling is hard._

The more users and requests you have, the harder it gets to maintain fast, consistent responses. Serving LLMs is one thing, but scaling LLMs brings its own challenges. Here are a few:

- When demand spikes, you need to make more resources available quickly. But dynamically scaling up GPU-bound workloads without dropping requests or disrupting service is complex.
- You can’t just send incoming requests to any available server. If someone is "chatting" with a model, there could be session context that gets lost. That makes load balancing more complicated than in stateless systems.
- Spinning up a new model isn’t instant. Large models can take minutes to load into GPU memory, and that might affect user experience.

## Enter **Fast Inference**: vLLM

With an appreciation for the hurdles LLM inferencing brings, it's easier to see why so many new tools and techniques are emerging to address those challenges.

Currently, one of the most impactful is [vLLM](https://github.com/vllm-project/vllm),  an open-source LLM inference engine that improves memory efficiency and throughput, the rate at which a model can process input requests or generate output over time ([learn about LLM inference performance metrics like throughput here](https://symbl.ai/developers/blog/a-guide-to-llm-inference-performance-monitoring/)).

### What makes vLLM fast?

There are several innovations under the hood that help vLLM achieve high performance:

- **[PagedAttention](https://arxiv.org/abs/2309.06180)**: vLLM uses this technique which rethinks how memory is allocated for the attention mechanism, drastically reducing memory fragmentation and overhead. Instead of storing every token's key/value cache in contiguous blocks, PagedAttention lets the model store these in virtual memory “pages,” allowing more requests to run concurrently without running out of GPU memory.

- **[Continuous Batching](https://docs.modular.com/glossary/ai/continuous-batching)**: vLLM can dynamically batch incoming requests together — even if they arrive at different times. That means it can serve many users at once without waiting for batch windows to fill up, dramatically improving throughput without hurting latency.

- **Optimized CUDA Kernels**: vLLM includes optimized CUDA kernels to maximize performance on specific GPUs and other hardware. This means better efficiency, translating to cost savings.

So, with vLLM:

- You can serve more concurrent requests using fewer GPUs.
- You get significantly better throughput compared to other engines.
- You reduce memory waste, letting you deploy bigger models on the same hardware.

## Enter **Furious Scaling**: KServe

If vLLM is your high-performance car engine, then **[KServe](https://kserve.github.io/website/master/)** is like your pit crew. Built on [Kubernetes](https://kubernetes.io/), **KServe** is a powerful open-source platform for deploying, scaling, and managing models in production.

While vLLM optimizes how fast your model does its thing, KServe ensures it's accessible and running, under unpredictable workloads, across multiple model versions, and across the cloud.  It abstracts the complexity of model serving infrastructure while benefiting from all of Kubernetes’ resilience and scaling features.

### How does KServe help?

Some of its key capabilities include:

- **[Autoscaling](https://kubernetes.io/docs/concepts/workloads/autoscaling/)**: Automatically scale model instances up or down based on demand - even to zero when idle, which helps save on costs.
- **Multi-model management**: Deploy, update, and rollback different model versions with minimal downtime.
- **Advanced routing**: Automatically route traffic to new model versions for [canary testing](https://kserve.github.io/website/master/modelserving/v1beta1/rollout/canary/) or A/B deployments.
- **Observability**: Integrates with tools like Grafana for monitoring performance, latency, and error rates.

So, with KServe:

- You can run and manage multiple models at once, within the same system.
- You can scale those models up or down automatically — even to zero if no one’s using them.
- You get the benefits of Kubernetes-native features like scalability and fault-tolerance, so your services restart automatically if they crash and scale reliably across clusters.

## Putting It Together

So far, we’ve looked at some of the key challenges in LLM serving — from GPU limitations to scaling pains. And while there are several ways to scale LLM inference (e.g. [llm-d](https://llm-d.ai/),[vLLM Production Stack](https://docs.vllm.ai/projects/production-stack/en/latest/)), this post focuses on how vLLM and KServe fit in the landscape as a practical, open-source solution that’s Kubernetes-native, flexible, and approachable for beginners.

- vLLM makes inference fast and efficient, using techniques like PagedAttention and continuous batching to squeeze the most out of GPUs.

- KServe handles deployment, autoscaling, routing, and observability — making models production-ready.

### Hands-On with KServe and vLLM (on CPU)

You don't have to start with GPUs or complex stacks to play around with LLM serving. This walkthrough shows how you can get started and learning by running a small model on a local Kubernetes cluster using KServe and vLLM on your own laptop.

This example will use the following:

- [kind](https://kind.sigs.k8s.io/): To create a local Kubernetes cluster.
- KServe Quickstart: A minimal configuration to get started with KServe.
- Hugging Face model server: A KServe runtime that uses vLLM as a backend under the hood (more on this later.)
- [facebook/opt-125m](https://huggingface.co/facebook/opt-125m): a lightweight transformer model that can run on CPU.

#### Creating your local cluster with KServe

Start by following the first page of the [KServe Quickstart](https://kserve.github.io/website/0.15/get_started/#install-the-kserve-quickstart-environment). It'll help you easily install tools like `kind`, `kubectl`, and `helm` and run the installation script which will automatically set up KServe's control plane, webhooks, ingress, and out-of-the-box serving runtimes to deploy models. A **ServingRuntime** defines how models of a certain format are served: which container image to use, what arguments or environment variables are needed, and what model formats are supported.

KServe provides several built-in runtimes like:

- tritonserver → TensorFlow, ONNX, PyTorch, TensorRT
- sklearnserver → SKLearn
- huggingfaceserver → HuggingFace transformers

This demo will focus on the HuggingFace runtime, which for text generation [uses vLLM by default](https://kserve.github.io/website/latest/modelserving/v1beta1/llm/huggingface/).

#### Deploying a model

With a cluster set up, you can deploy a model using a KServe InferenceService. This custom resource definition is the main interface that KServe uses for managing models and represents the model's logical endpoint for serving inferences. Below, we'll deploy the `facebook/opt-125m` model. We pass [runtime arguments](https://kserve.github.io/website/latest/modelserving/v1beta1/llm/huggingface/#hugging-face-runtime-arguments) pointing to the model hosted on Hugging Face. We also specify the Key-Value cache space through `VLLM_CPU_KVCACHE_SPACE`, from [vLLM's CPU runtime variables](https://docs.vllm.ai/en/latest/getting_started/installation/cpu.html#related-runtime-environment-variables). Setting it larger allows vLLM run more requests in parallel, which can hinder our CPU performance. Depending on your own constraints, you may have to play around the resource request and limits.

```bash
kubectl apply -f - <<EOF
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: opt-125
spec:
  predictor:
    model:
      modelFormat:
        name: huggingface
      args:
        - --model_name=opt-125
        - --model_id=facebook/opt-125
      image: kserve/huggingfaceserver:v0.15.0
      env:
        - name: VLLM_CPU_KVCACHE_SPACE
          value: "1"
      resources:
        requests:
          cpu: "4"
          memory: "4Gi"
        limits:
          cpu: "4"
          memory: "4Gi"
EOF
```

If you watch the logs of your ServingRuntime Pod, you'll see the vLLM server spinning up behind the scenes:

```bash
kubectl get pods -l serving.kserve.io/inferenceservice=opt-125
kubectl logs <POD_NAME> -c kserve-container --follow
```

It should look something like:

```bash
INFO 06-18 22:53:05 [__init__.py:256] Automatically detected platform cpu.
2025-06-18 22:53:12.550 1 kserve INFO [model_server.py:register_model():395] Registering model: opt-125
INFO 06-18 22:53:37 [config.py:583] This model supports multiple tasks: {'generate', 'reward', 'score', 'embed', 'classify'}. Defaulting to 'generate'.
WARNING 06-18 22:53:37 [_logger.py:72] device type=cpu is not supported by the V1 Engine. Falling back to V0.
WARNING 06-18 22:53:37 [_logger.py:72] uni is not supported on CPU, fallback to mp distributed executor backend.
INFO 06-18 22:53:37 [llm_engine.py:241] Initializing a V0 LLM engine (v0.8.1) with config: model='facebook/opt-125m', speculative_config=None, tokenizer='facebook/opt-125m', skip_tokenizer_init=False, tokenizer_mode=auto, revision=None, override_neuron_config=None, tokenizer_revision=None, trust_remote_code=False, dtype=torch.float16, max_seq_len=2048, download_dir=None, load_format=auto, tensor_parallel_size=1, pipeline_parallel_size=1, disable_custom_all_reduce=False, quantization=None, enforce_eager=True, kv_cache_dtype=auto,  device_config=cpu, decoding_config=DecodingConfig(guided_decoding_backend='xgrammar', reasoning_backend=None), observability_config=ObservabilityConfig(show_hidden_metrics=False, otlp_traces_endpoint=None, collect_model_forward_time=False, collect_model_execute_time=False), seed=None, served_model_name=facebook/opt-125m, num_scheduler_steps=1, multi_step_stream_outputs=True, enable_prefix_caching=None, chunked_prefill_enabled=False, use_async_output_proc=False, disable_mm_preprocessor_cache=False, mm_processor_kwargs=None, pooler_config=None, compilation_config={"splitting_ops":[],"compile_sizes":[],"cudagraph_capture_sizes":[256,248,240,232,224,216,208,200,192,184,176,168,160,152,144,136,128,120,112,104,96,88,80,72,64,56,48,40,32,24,16,8,4,2,1],"max_capture_size":256}, use_cached_outputs=False,
...
```

From there, check out the status of your InferenceService.

```bash
kubectl get isvc
```

```bash
NAME   URL                              READY   PREV   LATEST   PREVROLLEDOUTREVISION   LATESTREADYREVISION   AGE
opt    http://opt-125.default.example.com   True           100                              opt-125-predictor-00001   3m30s
```

You can describe the InferenceService to get descriptive status information too. If you checked it before it was ready, you will see useful information for debugging such as a `Waiting for runtime Pod to become available` message.

#### Making an inference request

There are a few ways to [go about it](https://kserve.github.io/website/master/get_started/first_isvc/#4-determine-the-ingress-ip-and-ports), but here's a simple way:

Port forward the service:

```bash
INGRESS_GATEWAY_SERVICE=$(kubectl get svc --namespace istio-system --selector="app=istio-ingressgateway" --output jsonpath='{.items[0].metadata.name}')
kubectl port-forward --namespace istio-system svc/${INGRESS_GATEWAY_SERVICE} 8080:80
```

And in a new terminal, send a request:

```bash
export INGRESS_HOST=localhost
export INGRESS_PORT=8080
export SERVICE_HOSTNAME=$(kubectl get inferenceservice opt-125 -o jsonpath='{.status.url}' | cut -d "/" -f 3)

curl -v http://${INGRESS_HOST}:${INGRESS_PORT}/openai/v1/completions \
-H "content-type: application/json" -H "Host: ${SERVICE_HOSTNAME}" \
-d '{
  "model": "opt",
  "prompt": "One thing about Denver is ",
  "stream": false,
  "max_tokens": 30,
  "stop": "."
}'
```

After a few seconds, you should receive a response:

```bash
{
  "id": "cmpl-a36be1a1-91ef-4b44-b9c6-f7aa5d8b84bc",
  "object": "text_completion",
  "created": 1750288691,
  "model": "opt-125",
  "choices": [
    {
      "index": 0,
      "text": " you are free to have different opinions",
      "logprobs": null,
      "finish_reason": "stop",
      "stop_reason": ".",
      "prompt_logprobs": null
    }
  ],
  "usage": {
    "prompt_tokens": 7,
    "total_tokens": 15,
    "completion_tokens": 8,
    "prompt_tokens_details": null
  }
}
```

### Wrapping Up

This hands-on example is specifically designed to be accessible, allowing you to experiment with LLM serving concepts on your local machine without the need for expensive GPUs or complex cloud setups.

By standing up a simple KServe cluster with a small model running on CPU, you’ve peeled back the curtain on how modern LLM inference systems actually work. You’ve seen how:

vLLM serves text generation requests behind an OpenAI-compatible API

KServe abstracts away Kubernetes complexity with declarative model deployment

Concepts like InferenceService and ServingRuntime connect infrastructure to model logic

This local setup gives you the space to:

- **Dive deeper** into LLM serving without needing expensive GPUs or cloud clusters

- **Observe** the behavior of real serving runtimes like vLLM in action

- **Learn** how scalable LLM inference works under the hood

### Conclusion

### Further Reading & Next Steps

Explore vLLM:

- [vLLM GitHub repo](https://github.com/vllm-project/vllm)
- [PagedAttention paper](https://arxiv.org/abs/2309.06180)
- [vLLM's Slack channel](https://inviter.co/vllm-slack)

Learn more about KServe:

- [KServe GitHub repo](https://github.com/kserve/kserve)
- [Using the vLLM backend](https://kserve.github.io/website/latest/modelserving/v1beta1/llm/huggingface/text_generation/#serve-the-hugging-face-llm-model-using-vllm-backend)
- [KServe's Slack channel](https://www.kubeflow.org/docs/about/community/)
