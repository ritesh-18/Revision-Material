# Chapter 34 — SRE for AI/ML Systems

## 34.1 Concept Explanation

AI/ML systems break differently from web services. Models drift. GPUs are scarce. Inference is expensive and slow. Outputs are nondeterministic. Datasets are massive. Failures often look like quality regressions, not outages.

This chapter covers SRE for AI/ML specifically. Many practices from earlier chapters apply; this is what changes.

The companion is the GenAI Full Stack Handbook also in this repository, which goes deeper on architectures. This chapter is the SRE perspective.

## 34.2 What Makes AI Systems Different

**Non-determinism.** Same input may produce different outputs. Caching is complex. Testing is statistical.

**Quality drift.** A model's outputs degrade as the world changes (data drift) or as the model itself is updated. Outages can be silent.

**Resource scarcity.** GPUs are expensive and limited. Auto-scaling is slower. Cold starts are minutes, not seconds.

**Cost volatility.** A user with a large input or an agent loop can cost 100x a normal request.

**New failure modes.** Hallucination. Prompt injection. Tool abuse. Bias. Safety violations.

**Black-box debugging.** Models are opaque; root cause is often "the model produced wrong output" without an actionable mechanism.

## 34.3 SLIs for AI Systems

Standard SLIs (latency, errors, availability) still apply. New ones to add:

**Time-to-first-token (TTFT).** Perceived latency in chat. Critical for UX.

**Tokens per second** during decode.

**Hallucination rate** (via LLM-as-judge or human review of a sample).

**Output schema validity** when structured outputs expected.

**Tool call success rate** for agents.

**Refusal rate** — when the model declines.

**Eval scores** rolling over time.

**Cost per resolved task** — the business-aligned metric.

## 34.4 GPU Operations

GPUs require SRE specialization:

**Scheduling.** Kubernetes with NVIDIA device plugin. MIG for partitioning A100/H100.

**Cold starts.** Loading 70B model takes minutes. Keep warm replicas.

**Right-sizing.** H100 is overkill for 7B models; use L40S or A10G.

**Multi-tenancy.** GPUs are expensive; bin-pack workloads where possible.

**Spot.** GPU spot is rare; reserved is common.

**Capacity planning.** Top GPUs are rationed. Reserve in advance.

## 34.5 Inference Servers

vLLM, TGI, TensorRT-LLM, Triton, SGLang. Each is a service the SRE operates.

Key SRE concerns:
- KV cache utilization.
- Continuous batching parameters.
- Quantization quality vs speed tradeoffs.
- GPU memory headroom.
- Queue depth and admission control.
- Model version rollouts.

Inference servers are stateful (KV cache). Treat them like databases more than like web servers.

## 34.6 RAG System Reliability

RAG = LLM + retrieval. Both can fail.

SRE concerns for RAG:
- Vector DB performance (latency, throughput, recall).
- Embedding pipeline freshness.
- Reranking latency.
- Index size and growth.
- Cross-tenant isolation.
- Eval against golden questions on a schedule.

When RAG quality degrades, the cause is often retrieval (wrong chunks) not generation (wrong model output). Trace through the pipeline.

## 34.7 Agent System Reliability

Agents add loops and tools, making operation harder.

SRE concerns for agents:
- Step limits (prevent infinite loops).
- Cost caps per session.
- Tool error rates.
- Workflow engine reliability (Temporal, etc.).
- Failed agent detection.
- Trace visibility per agent step.

Agents are the worst-case for cost runaway. Strict limits are non-negotiable.

## 34.8 The Eval Pipeline

The most important new SRE artifact: continuous evaluation.

Architecture:
1. Curated golden dataset of inputs.
2. Run inputs through current model + prompt.
3. Score outputs (LLM-as-judge or human).
4. Compare to baseline.
5. Alert on regressions.

Eval pipelines catch quality issues before users notice. Without them, AI systems degrade silently.

## 34.9 Model Version Management

Models, prompts, retrieval configs, and tools all version together.

Practices:
- Pin model versions explicitly (avoid `gpt-4` alias, use specific version).
- Version prompts in code or a registry.
- Eval before promoting any version.
- Canary new versions.
- Auto-rollback on eval regression.

A common failure: provider updates a model under the same name. Behavior changes overnight. Always pin.

## 34.10 Cost Engineering for AI

AI cost dominates production budgets at scale. SRE owns:
- Per-feature cost tracking.
- Per-user / per-tenant budgets.
- Model routing (cheap models for easy, frontier for hard).
- Caching (prefix, semantic).
- Token budgets per request.
- Context length controls.

A 10x cost difference between optimized and naive AI services is common.

## 34.11 Safety as an Operational Concern

Safety failures are SRE incidents:
- Harmful content generated.
- PII leaked in outputs.
- Tool abuse (agent emails customer data).
- Prompt injection successful.

Defenses:
- Output filters.
- Pre-screen tool inputs.
- Audit logs on tool calls.
- Kill switches per feature.

## 34.12 Observability for AI

Beyond standard metrics:
- LangSmith, Phoenix, Langfuse for prompt-level tracing.
- Track every LLM call (input tokens, output tokens, cost, model).
- Trace RAG retrieval and reranker scores.
- Per-feature cost dashboards.
- Eval scores over time.

Tools mature rapidly; pick one and standardize.

## 34.13 Multi-Tenancy

AI products are often multi-tenant. Concerns:
- Per-tenant data isolation in retrieval indices.
- Cost attribution per tenant.
- Per-tenant rate limits.
- Noisy neighbor isolation.

Per-tenant SLOs and budgets are increasingly standard.

## 34.14 Incident Patterns for AI

Common AI incidents:

**Cost spike.** Often a viral feature, an agent loop, or a long input bug. Detect via cost dashboard; mitigate with per-user caps.

**Quality regression.** Provider updated model, prompt change went wrong, RAG corpus changed. Detect via eval; mitigate via rollback.

**Inference outage.** vLLM crash, GPU failure, OOM. Detect via standard metrics; mitigate with replicas and failover.

**Capacity exhaustion.** Surge exceeded GPU pool. Detect via queue depth; mitigate with rate limits and degraded mode.

**Safety violation.** Harmful output reported. Detect via monitoring; mitigate with output filter or kill switch.

## 34.15 Production Architecture (AI SRE)

```
   Edge / CDN
        |
   API gateway with auth + per-user rate limits
        |
   LLM gateway (model routing, caching, observability)
        |
   +-- Self-hosted vLLM cluster (large models on H100s)
   +-- Self-hosted vLLM cluster (small models on A10G)
   +-- Provider API (frontier model fallback)
   +-- Embedding service
   +-- Reranker
   +-- Vector DB
        |
   Eval pipeline runs nightly + per-PR
        |
   Cost dashboards per feature, per tenant
        |
   Safety monitoring (toxicity, PII, injection)
        |
   Kill switches at LLM gateway level
```

## 34.16 Tradeoffs

| Choice | Win | Cost |
|---|---|---|
| Self-hosted models | Cheaper at scale | GPU ops |
| Provider APIs | Easy | Per-token cost |
| Strict cost caps | Predictable | Some user friction |
| No caps | UX flexibility | Cost runaway risk |
| Continuous eval | Catches drift | Compute cost |
| Eval only manual | Cheap | Slow drift detection |

## 34.17 Scaling Challenges

- GPU capacity globally constrained.
- KV cache memory limits concurrency.
- Eval cost scales with dataset and model size.
- Multi-region GPU placement.

## 34.18 Security

- Prompt injection defenses.
- Tool permission scoping.
- PII redaction in logs.
- Output filters.
- Audit logs on every model call.

Chapter 20 of the GenAI handbook covers AI security depth.

## 34.19 Deployment Guide

Operationalizing AI:
1. Adopt OpenTelemetry + an AI-specific observability tool.
2. Build eval pipeline.
3. Standardize on an inference server.
4. Implement model and prompt versioning.
5. Set up cost dashboards per feature.
6. Define safety monitoring.
7. Build kill switches.
8. On-call rotation for AI-specific incidents.

## 34.20 Monitoring Strategy

- TTFT, decode rate.
- Token volume.
- Cost per feature.
- Eval scores (continuous).
- Safety event count.
- Tool error rate.
- GPU utilization.
- KV cache fill.

## 34.21 Cost Optimization

- Model routing.
- Caching at all layers.
- Quantization.
- Right-sized GPUs.
- Per-user budgets.
- Context trimming.
- Spot for batch workloads.

## 34.22 Interview Questions

- *What's different about SRE for AI?*
- *Walk through observability for an LLM service.*
- *How do you handle a cost spike from an AI feature?*
- *Eval pipeline architecture?*
- *GPU capacity planning?*

## 34.23 Hands-on Exercises

1. List SLIs you would define for an LLM chat product.
2. Plan a continuous eval pipeline for one model + prompt.
3. Design cost monitoring per tenant for an AI feature.

## 34.24 Common Mistakes

- No eval pipeline; quality drifts silently.
- Unpinned model versions.
- No cost caps; spike destroys budget.
- Standard auto-scaling for GPU services (too slow).
- Treating AI like a stateless web service.

## 34.25 Enterprise Best Practices

Dedicated AI platform SRE function. Continuous eval as CI. Model and prompt versioning. Per-feature cost ownership. Safety monitoring central. Kill switches at every layer. Engineering reviews for new AI features include reliability and cost analysis. Quarterly review of model and inference stack as new options emerge.
