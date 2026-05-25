# Chapter 14 — GPU Systems

## 14.1 Concept Explanation

GPUs (Graphics Processing Units) are the engine of modern AI. They are not faster than CPUs in absolute terms — a CPU core typically clocks higher than a GPU core. They are *vastly more parallel*. A modern GPU has thousands of cores running thousands of threads in lockstep, optimized for the kind of parallel arithmetic that matrix multiplication requires.

The economics of GenAI are GPU economics. The bill, the latency, the scaling ceiling — all are downstream of GPU choice and utilization. An engineer who understands GPU architecture and operations can save organizations millions; one who does not, often costs them millions.

## 14.2 GPU Architecture Overview

A GPU has, conceptually:
- **Streaming Multiprocessors (SMs)** — the compute cores. NVIDIA H100 has 132. A100 has 108.
- **CUDA cores** — within each SM, the basic arithmetic units. Thousands per GPU.
- **Tensor cores** — specialized matrix-math units, much faster than CUDA cores for the operations LLMs care about. The reason LLMs run well on GPUs is tensor cores.
- **HBM (High Bandwidth Memory)** — the GPU's main memory. Fast, expensive, the limit on model size.
- **L2 cache and registers** — on-chip memory for staging data near the compute.

The metric that matters most for LLM inference: **memory bandwidth**, measured in TB/s. The H100 has 3.35 TB/s; A100 has 2 TB/s; consumer cards like RTX 4090 have around 1 TB/s. This is what feeds the compute cores.

## 14.3 CUDA Cores vs Tensor Cores

CUDA cores do general arithmetic — adds, multiplies, comparisons — on individual numbers. Tensor cores do whole matrix multiplications in a single operation, at much higher throughput.

For LLMs, almost all the FLOPs come from tensor cores. CUDA cores handle the surrounding logic. NVIDIA's marketing TFLOPs numbers usually quote tensor core performance; real-world inference performance depends on how well the inference server keeps the tensor cores fed.

## 14.4 VRAM and HBM

VRAM is the GPU's main memory. For LLMs, two questions dominate:
- Does the model fit?
- Does the model plus KV cache plus activations fit?

A rough sizing guide for inference in FP16/BF16:
- **7B model** — ~14 GB weights, plus ~4 GB KV cache for moderate concurrency. Fits on a single 24 GB consumer GPU.
- **13B model** — ~26 GB. Needs 40 GB A100 or H100, or 2x consumer GPU with tensor parallelism.
- **70B model** — ~140 GB. Needs multiple GPUs.
- **405B+ model** — multiple H100/H200 nodes with high-speed interconnect.

Quantization halves or quarters these numbers. INT8 70B fits in 80 GB; INT4 fits in around 40 GB.

## 14.5 NVLink, PCIe, and Interconnect

When multiple GPUs serve one model, they must exchange data at every layer. The bandwidth and latency of this exchange dictates whether multi-GPU is worth it.

- **PCIe Gen5 x16** — ~64 GB/s. Adequate for most workloads, bottlenecks heavy multi-GPU LLM inference.
- **NVLink 4.0 (H100)** — ~900 GB/s per GPU pair. Designed for multi-GPU model parallelism.
- **NVSwitch** — connects many NVLink endpoints into a fully connected fabric within a node.
- **InfiniBand / RoCE** — cross-node, ~400-800 Gbps. Used for multi-node training and large inference clusters.

The rule: if you are doing tensor parallelism, NVLink matters enormously. If you are doing per-replica scaling (independent GPUs serving the same model), PCIe is fine.

## 14.6 Multi-GPU Systems

A typical AI server node has 4 or 8 GPUs connected by NVLink/NVSwitch. NVIDIA's reference designs (HGX H100/H200) form the basis of most cloud GPU offerings.

Multi-node clusters connect dozens to thousands of these nodes via InfiniBand or high-speed Ethernet. Training a frontier model requires thousands of GPUs running synchronized for weeks. Inference at scale uses smaller pools — typically tens to hundreds of GPUs per region.

## 14.7 MIG — Multi-Instance GPU

A100 and H100 GPUs support MIG, which partitions a single GPU into up to seven smaller "instances," each with isolated memory and compute. This lets you serve multiple small models or multiple small workloads on one physical GPU without interference.

When to use: many small workloads (multiple embedding models, small fine-tunes), per-tenant isolation, dev environments. When not to use: large models that need the full GPU, latency-critical workloads (MIG instances are smaller than the parent GPU).

## 14.8 GPU Sharing in Kubernetes

Beyond MIG, several mechanisms share GPUs across pods:
- **Time-slicing** — GPUs round-robin between pods at the driver level. Simple, but unpredictable performance.
- **MIG** — hardware-isolated partitions, predictable performance.
- **MPS (Multi-Process Service)** — NVIDIA's CUDA-level sharing, allows multiple processes to issue CUDA kernels simultaneously to one GPU.
- **vGPU** — NVIDIA's virtualization, common in VDI; less common in AI.

The NVIDIA Device Plugin and GPU Operator for Kubernetes expose these as schedulable resources. Chapter 15 covers cluster deployment.

## 14.9 Inference Optimization Techniques

Beyond the inference servers in Chapter 13, several techniques live at the GPU level:

- **Mixed precision (FP16, BF16, FP8).** Modern GPUs run these much faster than FP32.
- **Operator fusion.** Combining multiple operations into one kernel reduces memory roundtrips.
- **CUDA graphs.** Pre-record a sequence of kernels and replay it with minimal CPU overhead.
- **FlashAttention.** A highly optimized attention kernel that reduces memory traffic; standard in modern inference servers.
- **TensorRT-LLM.** Compiles models into highly optimized engines for NVIDIA hardware.

Most of these are invisible to application engineers because the inference server bakes them in. But knowing they exist explains where new versions of vLLM and TGI get their performance jumps.

## 14.10 Quantization

Quantization reduces precision (Chapter 4 introduced this conceptually; here we cover hardware).

- **FP16 / BF16** — native training precision; minor accuracy loss is rare.
- **FP8** — supported on H100 and newer. Roughly 2× throughput vs FP16.
- **INT8** — well-supported across most GPUs; small accuracy loss; common in production.
- **INT4** — supported via AWQ, GPTQ, and other algorithms; 4× memory savings; noticeable but manageable accuracy loss.

Choose the most aggressive quantization that passes your evaluation set. Don't trust generic "INT4 is fine" claims for your specific task — always benchmark.

## 14.11 LoRA and QLoRA — Fine-Tuning on Modest GPUs

LoRA (Low-Rank Adaptation) freezes the base model and inserts small trainable "adapter" matrices. Only the adapters are trained, dramatically reducing memory and compute needs. A 70B model fine-tune that would require multiple H100s becomes feasible on a few A100s.

QLoRA combines LoRA with 4-bit quantization of the base model. This is what enables 70B fine-tunes on a single 24 GB consumer GPU (slowly, but at all).

LoRA adapters are tiny (typically 1-100 MB). You can serve a base model and switch between many LoRA adapters per request, enabling personalization at scale.

## 14.12 NVIDIA Stack — The De Facto Standard

NVIDIA's lock on AI is software as much as hardware.
- **CUDA** — the language and runtime.
- **cuDNN** — primitives for deep learning operations.
- **cuBLAS** — basic linear algebra.
- **NCCL** — multi-GPU communication.
- **TensorRT / TensorRT-LLM** — optimized inference engines.
- **Triton Inference Server** — production model serving.
- **Magnum IO** — data movement libraries.
- **NIM** — pre-packaged inference microservices.

Most AI frameworks compile to this stack. Replacing NVIDIA means replacing all of it.

## 14.13 AMD Stack

AMD's MI300X is a competitive GPU on paper (192 GB HBM3, strong throughput), and ROCm — AMD's CUDA equivalent — has improved dramatically. Many open-source inference servers now support AMD.

Strengths: more memory per GPU than current NVIDIA at similar tiers, often more available in cloud markets, sometimes better price per FLOP. Weaknesses: smaller ecosystem, less battle-tested at scale, fewer pre-trained kernels and operators.

The trajectory matters. AMD is the strongest credible challenger and many companies are evaluating it as a hedge against NVIDIA pricing.

## 14.14 Other AI Accelerators

- **Google TPUs** — Google's custom chips, available via GCP. Strong for training and Google's own inference.
- **AWS Inferentia / Trainium** — AWS custom chips. Lower cost per inference for compatible models, narrower model support.
- **Intel Gaudi** — Intel's AI accelerator line. Competitive on price; ecosystem is the gap.
- **Cerebras, Groq, SambaNova, Etched** — startups with novel architectures. Groq's deterministic, ultra-low-latency inference is notable for chat applications.
- **Apple Silicon** — Metal Performance Shaders, used by llama.cpp on Mac for surprisingly good local inference.

Most production teams stay on NVIDIA. Diversification is happening but slowly.

## 14.15 Cloud GPU Comparison

| Provider | Strengths | Weaknesses |
|---|---|---|
| AWS | Most regions, broad services | GPU supply tight, premium pricing |
| Azure | Tight OpenAI integration | Capacity constraints, complex pricing |
| GCP | TPUs as alternative, strong networking | Smaller GPU footprint than AWS |
| Oracle (OCI) | Aggressive GPU pricing, free egress | Smaller ecosystem |
| RunPod | Cheap on-demand and spot | Fewer enterprise features |
| Lambda Labs | Simple, AI-focused | Capacity volatility |
| CoreWeave | Large-scale GPU specialist | More opaque than hyperscalers |
| Crusoe, FluidStack, Together AI | Niche GPU offerings | Varied maturity |

For self-hosted inference at scale, secondary providers (CoreWeave, Lambda, RunPod, Crusoe) often offer 30-50% cost savings versus hyperscalers, at the cost of less enterprise tooling and capacity uncertainty.

## 14.16 Production Architecture

```
   Inference cluster (per region)
     |
     +--- Pool A: H100 nodes, large models, paid tier traffic
     +--- Pool B: A10G nodes, small models, free tier traffic
     +--- Pool C: L40S nodes, embedding and reranking
     +--- Pool D: spot capacity, batch workloads
     |
   Autoscaler (per pool)
     |
   GPU operator + device plugin (Kubernetes)
     |
   Observability (DCGM exporter, Prometheus, Grafana)
```

Multiple pools let you match GPU class to workload economics.

## 14.17 Tradeoffs

| Choice | Win | Cost |
|---|---|---|
| H100 | Top performance | Highest price, scarcest |
| A100 | Strong, mature | Older, less FP8 support |
| L40S / A10G | Cheap per replica | Smaller memory |
| Consumer (RTX 4090) | Cheapest | No NVLink, no datacenter support |
| MI300X | More memory, competitive perf | Software maturity lag |
| TPU | Strong on Google, Jax | Lock-in, niche tooling |

## 14.18 Scaling Challenges

- **GPU supply.** Top-end GPUs are rationed by cloud providers. Reserve capacity in advance.
- **Cold starts.** Loading a 70B model into GPU memory takes minutes. Keep warm replicas.
- **Multi-region.** GPU pricing varies; some regions have no high-end GPU availability.
- **Power and cooling.** Datacenter constraints affect availability and cost. This is a real bottleneck in some regions.

## 14.19 Security Concerns

- GPU memory can hold sensitive data; isolate tenants with separate processes or MIG.
- Side-channel risks exist between MIG instances under specific configurations.
- GPU firmware and driver versions must be kept current for security and stability.

## 14.20 Cost Optimization

- Use spot/preemptible GPUs for batch workloads (often 70% cheaper).
- Right-size GPU class to workload (don't run a 7B model on H100).
- Quantize aggressively and benchmark.
- Use LoRA adapters to multiplex personalization on shared base models.
- Negotiate reserved capacity at scale.
- Evaluate secondary providers for cost.

## 14.21 Interview Questions

- Walk through the memory layout of a 70B model in GPU VRAM.
- Compare H100, A100, and L40S for chat inference at scale.
- Explain MIG vs MPS vs time-slicing.
- When does tensor parallelism help and when does it hurt?
- Estimate cost per million tokens for hosting Llama 70B on H100.

## 14.22 Hands-on Exercises

1. Plan a GPU pool for 1M chat sessions per day on a 13B model.
2. Calculate cost per million tokens for self-hosted 70B at $4/H100/hour with measured throughput of 2,000 tokens/sec/GPU.
3. Compare three providers on price per H100-hour and design a multi-cloud failover.

## 14.23 Common Mistakes

- Buying H100s when L40S or A10G suffice.
- Ignoring memory bandwidth — TFLOPs alone do not predict LLM inference speed.
- Skipping quantization "to be safe" without measuring.
- Forgetting NVLink in multi-GPU planning.
- No warm capacity; cold starts cause user-visible delays.

## 14.24 Enterprise Best Practices

Treat GPU capacity as a strategic resource, not an operational one. Reserve frontier-GPU capacity in advance. Diversify across two providers for resilience. Maintain an internal benchmark suite that exercises your specific workloads on candidate GPUs. Plan annual GPU refresh cycles around new model launches and hardware generations.
