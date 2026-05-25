# Chapter 13 — AI Inference Systems

## 13.1 Concept Explanation

An inference system is the software that runs a trained model and serves predictions to clients. For LLMs, this is not "load model, call forward, return output" — that approach hits a wall above a single user. Production inference systems handle batching, KV cache management, parallelism, scheduling, and streaming, often producing 10–100× more throughput than naive serving.

This chapter goes deep on the systems that make modern LLM inference economically viable: vLLM, TGI, Triton, TensorRT-LLM, llama.cpp, SGLang, Ollama. Understanding them lets you choose, deploy, and operate inference clusters with confidence.

## 13.2 The Inference Workload Has Two Phases

LLM inference has fundamentally different characteristics in two phases:

**Prefill phase.** Processing the prompt (input tokens). All tokens are known up front, so they can be processed in parallel. Compute-bound. GPU FLOPs are the bottleneck.

**Decode phase.** Generating output tokens one at a time. Each token requires reading all model weights to produce the next prediction. Memory-bandwidth-bound. HBM throughput is the bottleneck.

A system optimized for one phase is suboptimal for the other. Modern inference servers schedule prefill and decode together so the GPU is always busy doing whatever it can do best.

## 13.3 vLLM — The Modern Standard

vLLM, from UC Berkeley, is the most widely adopted open-source LLM inference server. Its key innovation is **PagedAttention**: managing KV cache as virtual memory pages rather than contiguous blocks. This eliminates fragmentation, allows much higher batch sizes, and dramatically improves throughput.

Other features:
- Continuous batching (also called in-flight batching).
- Tensor parallelism for multi-GPU inference.
- Speculative decoding.
- Prefix caching for repeated prompts.
- Compatibility with Hugging Face model formats.
- Production-grade OpenAI-compatible API.

vLLM is the default starting point for self-hosted open-source models in production today.

## 13.4 TGI — Hugging Face Text Generation Inference

TGI is Hugging Face's inference server. Strengths: tight integration with Hugging Face Hub, broad model support, production-grade with built-in tracing and metrics. Slightly behind vLLM on raw throughput in many benchmarks but easier integration in HF-centric ecosystems.

TGI supports streaming, tensor parallelism, quantization, watermarking, and a clean Rust core. A solid choice when your organization is already on the Hugging Face ecosystem.

## 13.5 TensorRT-LLM — NVIDIA's High-Performance Path

TensorRT-LLM is NVIDIA's library for compiling LLMs into highly optimized TensorRT engines. The result is the fastest inference on NVIDIA hardware, period. Cost: build complexity, version sensitivity, and lock-in to NVIDIA's stack.

When to use: production inference on NVIDIA hardware where every millisecond and every dollar matters, and the engineering team can absorb the build complexity. Often used inside Triton Inference Server.

## 13.6 Triton Inference Server

Triton is NVIDIA's general-purpose inference server. It serves any model in any framework (PyTorch, TensorFlow, ONNX, TensorRT-LLM, vLLM as a backend) behind a unified API. Strong scheduling, dynamic batching, model ensembles, multi-GPU.

Triton is the right answer when you serve many different models, or when you want enterprise-grade orchestration plus deep NVIDIA optimization. It is more complex than vLLM but more general.

## 13.7 llama.cpp

llama.cpp is the open-source C++ project that runs LLMs efficiently on CPUs and consumer GPUs with aggressive quantization. It is what makes running 70B models on a MacBook possible.

Strengths: minimal dependencies, runs anywhere, GGUF quantization formats, broad hardware support (CUDA, Metal, ROCm, Vulkan, CPU).

Limits: not designed for high-throughput multi-user serving. Use for local development, on-device inference, or low-traffic deployments.

## 13.8 SGLang

SGLang is a newer inference engine focused on structured generation, complex prompting patterns, and high throughput. Strong at workloads with branching, JSON-constrained outputs, and prefix sharing across many requests. Used by teams running heavily structured agent workloads.

## 13.9 Ollama

Ollama is a wrapper around llama.cpp focused on developer ergonomics — a single binary, a simple CLI, a local API. Great for prototyping, local dev, and on-device deployments. Not for high-traffic production.

## 13.10 Comparison Table

| Server | Best for | Throughput | Multi-GPU | Hardware |
|---|---|---|---|---|
| vLLM | Open-source production | Very high | Yes | NVIDIA + AMD |
| TGI | HF ecosystem | High | Yes | NVIDIA + AMD |
| TensorRT-LLM | Max NVIDIA perf | Highest on NVIDIA | Yes | NVIDIA only |
| Triton | Multi-model serving | High | Yes | NVIDIA + others |
| llama.cpp | Local / on-device | Low | Limited | Wide |
| SGLang | Structured/agent | Very high | Yes | NVIDIA |
| Ollama | Dev / local | Low | Limited | Wide |

## 13.11 Continuous Batching

Naïve batching gathers a fixed number of requests, processes them together, returns all results. This wastes capacity when individual requests finish at different times.

Continuous batching (in-flight batching) constantly admits new requests and evicts completed ones at every step. The result: GPU stays full, latency stays low, throughput climbs. Every modern inference server does this.

## 13.12 KV Cache Management

The KV cache (Chapter 4) holds per-user state. Managing it well is the difference between 10 and 100 concurrent users on the same GPU.

PagedAttention (vLLM) treats KV cache as fixed-size blocks managed like virtual memory pages. Requests share the page table. Fragmentation drops, packing improves, throughput rises.

Prefix caching detects when multiple requests share a common prefix (the system prompt, a shared retrieval context) and reuses the KV computation for that prefix. Throughput gain depends on prefix overlap; for chatbots with stable system prompts, the gain is large.

## 13.13 Tensor Parallelism

Tensor parallelism splits each weight matrix across multiple GPUs. The matrix multiplications are split too; partial results are exchanged via NCCL all-reduce. This lets a model that does not fit in one GPU's memory be served across two or four GPUs.

Cost: inter-GPU communication during every forward pass. NVLink between GPUs makes this fast; PCIe makes it slow. The hardware matters.

Tensor parallelism is most useful for very large models (70B and up). Below that, it often hurts throughput because communication overhead dominates.

## 13.14 Pipeline Parallelism

Pipeline parallelism splits the model by layer across GPUs — GPU 0 holds layers 1-20, GPU 1 holds 21-40, etc. Each batch is split into "micro-batches" that flow through the pipeline so GPUs are not all waiting for one.

Pipeline parallelism scales further than tensor parallelism but is harder to make latency-friendly. Common at extreme scale (huge models across many nodes) and rare at single-node scale.

## 13.15 Speculative Decoding

A small "draft" model proposes several tokens, the large "target" model verifies them in parallel. When the draft agrees with the target (most of the time on easy tokens), generation speeds up by 2–5× with no quality loss.

Implementation matters: the draft model must share tokenizer with the target. Recent work uses the target model itself in a self-speculative configuration, eliminating the need for a separate draft.

## 13.16 Throughput vs Latency Tradeoff

Bigger batches = higher throughput but higher per-request latency. The optimal point depends on the workload.

- **Latency-sensitive chat.** Small batches, prioritize time-to-first-token, possibly use speculative decoding.
- **Throughput-oriented batch processing.** Large batches, accept higher per-request latency.
- **Mixed workload.** Continuous batching with priority queues. Latency-critical requests jump the queue.

## 13.17 GPU Scheduling

Within a single GPU, the scheduler decides which requests to admit, which to preempt, which to evict from KV cache when memory is tight.

Modern schedulers consider:
- Available KV cache memory.
- Request priority (free tier vs paid tier).
- Estimated remaining tokens per request.
- Fairness across users.
- Deadline awareness (streaming chat needs first token fast; offline batch can wait).

vLLM and TensorRT-LLM both have sophisticated schedulers with knobs to tune.

## 13.18 Inference Pipeline End-to-End

```
   Request
      |
   Tokenizer
      |
   Scheduler decides admission
      |
   Prefill (parallel, compute-bound)
      |
   KV cache populated
      |
   Decode loop (memory-bandwidth-bound)
      |   per step:
      |     - run forward through cached K/V
      |     - sample next token
      |     - stream token to client
      |     - append to KV cache
      |
   Decode finishes (EOS, max tokens, or stop sequence)
      |
   KV cache released
      |
   Final token sent, stream closed
```

## 13.19 Production Architecture

```
   Client
      |
   API gateway (auth, rate limit)
      |
   LLM router / gateway (LiteLLM, Portkey)
      |
   +--- vLLM cluster (model A)
   +--- vLLM cluster (model B, quantized)
   +--- External provider (frontier model fallback)
   +--- Embedding server (TEI or vLLM)
   +--- Reranker server
      |
   Observability and tracing
```

Each model cluster is autoscaled separately based on its own load and characteristics.

## 13.20 Tradeoffs

| Choice | Win | Cost |
|---|---|---|
| vLLM | Open standard, best perf/setup ratio | New version cycles |
| TensorRT-LLM | Max perf on NVIDIA | Build complexity, lock-in |
| llama.cpp | Run anywhere | Low throughput |
| Tensor parallelism | Fits big models | NVLink dependence |
| Speculative decoding | 2-5x speedup | Setup complexity |

## 13.21 Scaling Challenges

- **Cold starts.** Loading a 70B model into GPU memory takes minutes. Keep warm replicas.
- **Memory fragmentation.** Long-running servers fragment KV cache; periodic restarts can help.
- **Tail latency.** A few requests with very long inputs stall others. Use admission control and per-request token budgets.
- **Multi-region.** Replicating GPU clusters across regions is expensive; route latency-sensitive traffic by region.

## 13.22 Security Concerns

- Inference servers should not be exposed directly to the internet. Place behind a gateway with auth and rate limiting.
- Validate input lengths to prevent prompt-flood DoS.
- Separate tenants either by separate clusters or by careful KV cache isolation.

## 13.23 Deployment Guide

Containerize the inference server (vLLM publishes images). Deploy on Kubernetes with the NVIDIA device plugin and GPU operator. Use autoscalers that consider GPU utilization, queue depth, and per-request token budgets, not just CPU. Chapter 15 details deployment.

## 13.24 Monitoring Strategy

- p50/p95/p99 time-to-first-token.
- p50/p95/p99 time-to-last-token.
- Tokens per second per request and aggregate.
- KV cache utilization.
- Queue depth and admission rate.
- GPU memory and utilization.
- Cost per million tokens.
- Error rate by type (OOM, timeout, abort).

## 13.25 Cost Optimization

- Pick the smallest model that passes evals.
- Quantize (AWQ, GPTQ, FP8) for memory and speed wins.
- Use speculative decoding where applicable.
- Right-size GPUs (A10G vs H100 vs L40S) for the workload.
- Use spot instances for batch workloads.

## 13.26 Interview Questions

- Walk through PagedAttention.
- Compare tensor parallelism and pipeline parallelism.
- Why is decode memory-bandwidth-bound while prefill is compute-bound?
- Design an inference cluster for 10,000 QPS chat traffic.
- How would you reduce TTFT by 30%?

## 13.27 Hands-on Exercises

1. Estimate the GPU memory required to host a 70B model in FP16 with KV cache for 64 concurrent users at 8K context.
2. Plan a deployment that serves three model sizes (small, medium, large) for different query difficulties.
3. Sketch the autoscaling policy for a vLLM cluster facing bursty chat traffic.

## 13.28 Common Mistakes

- Naïve batching at fixed sizes.
- Ignoring prefix caching for stable system prompts.
- Single-replica clusters with no warm capacity.
- Autoscaling on CPU instead of GPU and queue depth.
- Treating prefill and decode latency as the same metric.

## 13.29 Enterprise Best Practices

Standardize on an inference server per model family. Build an internal "LLM gateway" that abstracts which backend serves which request. Run quarterly performance reviews to evaluate new versions of vLLM/TGI/TensorRT. Maintain per-model latency and throughput SLAs. Plan for new hardware generations as part of cost forecasting.
