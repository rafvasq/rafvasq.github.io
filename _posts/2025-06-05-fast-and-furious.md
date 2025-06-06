---
layout: post
title:  "Fast Inference and Furious Scaling with vLLM and KServe"
date:   2025-06-05
categories: vllm kserve
---

_Why serving large language models is hard — and how vLLM + KServe can help beginners get started._

The name "Fast Inference, Furious Scaling", obviously inspired by the movies, is not just catchy, but also captures the atmosphere of this area of technology. With a constant influx of tweets, research papers, and code outlining new methodologies and optimization algorithms, and not enough "good first issues", this space moves at a pace that's hard to *enter into*, let alone keep up with.

In this post, beginners who are new to the world of LLM inferencing and serving can learn about why its complicated thing to do and gain a clearer idea of how to get started with it using two open source tools: vLLM and KServe. Instead of bogging down in technical details, the focus is on the 'why' and 'how' to give a background context for those who want to participate, while including resources for further deep-diving linked along the way.

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

The more users and requests you have, the harder it gets to maintain fast, consistent responses. Serving LLMs is one thing, but scaling LLMs brings its own challenges:

- **Autoscaling**: dynamically adjusting resources without disruption is challenging on stateful, GPU-bound workloads.
- **Load balancing**: Distributing user requests to the right model instance isn't simple. A user might be mid-conversation with a model, so their next request needs to go to a server that has their context in memory.
- **Cold starts**: Launching new instances can take minutes, especially with large models that need to be loaded into GPU memory. During that time, users may experience latency spikes or dropped requests.

## Enter **Fast Inference**: vLLM

With an appreciation for the hurdles LLM inferencing brings, it's easier to see why so many new tools and techniques are emerging to address those challenges. One of the most impactful is **vLLM**.

Originally developed at UC Berkeley, [vLLM is a community-driven open-source LLM inference engine](https://github.com/vllm-project/vllm) working to improve LLM memory efficiency and **throughput**, the rate at which a model can process input requests or generate output over time.

[You can learn about LLM inference performance metrics here](https://symbl.ai/developers/blog/a-guide-to-llm-inference-performance-monitoring/).

### What makes vLLM fast?

There are several innovations under the hood that help vLLM achieve high performance:

- **[PagedAttention](https://arxiv.org/abs/2309.06180)**: vLLM uses this technique which rethinks how memory is allocated for the attention mechanism, drastically reducing memory fragmentation and overhead. Instead of storing every token's key/value cache in contiguous blocks, PagedAttention lets the model store these in virtual memory “pages,” allowing more requests to run concurrently without running out of GPU memory.

- **[Continuous Batching](https://insujang.github.io/2024-01-07/llm-inference-continuous-batching-and-pagedattention/)**: vLLM can dynamically batch incoming requests together — even if they arrive at different times. That means it can serve many users at once without waiting for batch windows to fill up, dramatically improving throughput without hurting latency.

- **Optimized CUDA Kernels**: vLLM includes optimized CUDA kernels to maximize performance on specific GPUs and other hardware. This means better efficiency, translating to cost savings.

So, with vLLM:

- You can serve more concurrent requests using fewer GPUs.
- You get significantly better _throughput_ compared to naive implementations.
- You reduce memory waste, letting you deploy bigger models on the same hardware.

## Enter **Furious Scaling**: KServe

If vLLM is your high-performance car engine, then **KServe** is like your pit crew. Built on [Kubernetes](https://kubernetes.io/), **KServe** is a powerful open-source platform for deploying, scaling, and managing models in production.

Where vLLM optimizes how fast your model does its thing, KServe ensures its accessible and running — even under unpredictable workloads, across multiple model versions, and across the cloud.  It's meant to abstract the complexity of model serving infrastructure while benefiting from all of Kubernetes’ resilience and scaling features.

### How does KServe help?

Some of its key capabilities include:

- **Autoscaling**
    Automatically scale model instances up or down based on real-time traffic, including support for scaling to zero when idle — reducing unnecessary compute spend.
- **Multi-model management**
    Deploy, update, and rollback different model versions with minimal downtime.
- **Explainer and Transformer integration**
    KServe can run model explainers or input/output preprocessors alongside the main model.
- **Advanced routing & traffic splitting**
    Automatically route traffic to new model versions for canary testing or A/B deployments.
- **Built-in observability**
    Exposes Prometheus metrics and integrates with tools like Grafana for monitoring performance, latency, and error rates.
