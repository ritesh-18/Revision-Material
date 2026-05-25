# Chapter 19 — Monitoring and Observability

## 19.1 Concept Explanation

Observability is the property that lets you ask new questions about a running system without redeploying it. The traditional pillars are metrics, logs, and traces. GenAI adds three more: evaluations, costs, and safety signals.

A GenAI system without observability is a dare. You will not know which model version is degrading, which prompt is hallucinating, which tenant is burning budget, which retrieval path is failing. The cost of building observability is small; the cost of not having it is invisible until it is enormous.

## 19.2 The Three Plus Three Pillars

```
   Traditional:
     Metrics  — numerical time series (latency, error rate, QPS)
     Logs     — text records of events
     Traces   — request paths across services

   GenAI additions:
     Evals    — quality scores against expected behavior
     Cost     — per-request and aggregate spend
     Safety   — refusals, harmful outputs, injection attempts
```

Each pillar has tools. Mature stacks treat all six as first-class.

## 19.3 Prometheus

Prometheus is the dominant open-source metrics system. It scrapes time-series data from instrumented services, stores it efficiently, and supports a powerful query language (PromQL).

For GenAI, scrape every service: app servers (latency, tokens), inference servers (KV cache, GPU utilization via DCGM exporter), vector DBs (query latency), gateways (request rate). Aggregate to dashboards and alerts.

Strengths: open source, ubiquitous, strong ecosystem. Weaknesses: long-term storage requires extensions (Thanos, Mimir, VictoriaMetrics), cardinality blow-up is easy.

## 19.4 Grafana

Grafana is the dominant visualization tool. Connect it to Prometheus, Loki, Tempo, and many other backends; build dashboards and alerts.

For GenAI: one dashboard per service, one dashboard per AI feature, one dashboard for the LLM gateway. The discipline is making each dashboard small and answerable at a glance.

## 19.5 Loki

Loki is Grafana Labs' log aggregation system. Indexes only metadata, stores log lines compressed in object storage. Cheap, scalable.

For GenAI, log structured events per request: trace ID, user ID, model, tokens, cost, latency, retrieval IDs, tool calls. Avoid logging full prompts and completions unless redacted or sampled — they balloon cost and risk PII.

## 19.6 ELK Stack

Elasticsearch + Logstash + Kibana — the classic enterprise logging stack. Powerful full-text search, rich aggregations, mature. More expensive than Loki at scale but better for complex investigative queries.

OpenSearch (the AWS-led fork) is functionally equivalent and Apache-licensed.

## 19.7 OpenTelemetry

OpenTelemetry is the open standard for instrumentation. One set of SDKs, one wire protocol, many backends (Prometheus, Tempo, Jaeger, Datadog, Honeycomb).

Adopt OpenTelemetry early. It decouples instrumentation from any specific vendor; you can swap backends without re-instrumenting your code.

For GenAI: emit a span for every LLM call with attributes (model, tokens, prompt hash, retrieval IDs, cost). The trace becomes a complete record of what the system did.

## 19.8 Jaeger and Tempo

Distributed tracing backends. Jaeger is the classic open-source choice; Tempo is Grafana Labs' newer, cheaper alternative (uses object storage).

Traces are the single most useful artifact when debugging a degraded request. You see exactly where time went, which service slowed, which retrieval missed, which tool failed.

## 19.9 LangSmith

LangSmith (from LangChain) is a specialized observability platform for LLM applications. It captures every LLM call with prompts, completions, latencies, costs, eval scores, and a UI tuned for prompt iteration.

Strengths: prompt-aware, integrated with LangChain and most major LLM SDKs, eval features. Weaknesses: yet another vendor, hosted SaaS by default.

## 19.10 Phoenix

Arize Phoenix is open-source observability and evaluation for LLMs and traditional ML. Strong on drift detection, retrieval evaluation, and embedding visualization.

## 19.11 Langfuse, Helicone, Portkey

Other LLM-focused observability tools:
- **Langfuse.** Open source, self-hostable, prompt management included.
- **Helicone.** Proxy-style observability — point your LLM client at Helicone, get full visibility.
- **Portkey.** Combines LLM gateway, caching, and observability.

These tools overlap with each other. Pick one and standardize.

## 19.12 What to Monitor — Application Metrics

- **Throughput.** Requests per second, by endpoint and model.
- **Latency.** p50, p95, p99 of total response time and time-to-first-token.
- **Error rate.** By type (timeouts, 5xx, model refusals, format failures).
- **Active connections.** Especially for streaming endpoints.
- **Cache hit rate.** Raw cache and semantic cache.
- **Token counts.** Input, output, total. By model, by feature, by tenant.

## 19.13 What to Monitor — Model Metrics

- **Time-to-first-token.** Most important perceived-latency metric.
- **Tokens per second** during decode.
- **Refusal rate** (model declined to answer).
- **Tool call rate** and tool call success rate.
- **Hallucination rate** — measured via LLM-as-judge or retrieval-grounded checks.
- **Output schema validation rate** for structured outputs.

## 19.14 What to Monitor — Cost

- **Cost per request, per user, per tenant, per feature.**
- **Cost per resolved task** — the business-meaningful denominator.
- **Cost by model.** Useful for routing decisions.
- **Cost trend.** Daily, weekly, monthly. Set anomaly alerts on sudden spikes.
- **Budget burn rate** vs allocated.

## 19.15 What to Monitor — Quality / Evals

- **Eval scores** by prompt version, model version, dataset.
- **User feedback rate** (thumbs up/down ratio).
- **Regeneration rate.**
- **Edit acceptance rate** (for code or content suggestions).
- **Citation accuracy rate** in RAG.

## 19.16 What to Monitor — Safety

- **Refusals** — both count and pattern.
- **Detected prompt injection attempts.**
- **PII detection events.**
- **Toxicity flags** from content classifiers.
- **Tool calls to sensitive endpoints** for human review.

## 19.17 What to Monitor — Infrastructure

- **GPU utilization** via NVIDIA DCGM exporter.
- **GPU memory** usage and fragmentation.
- **Inference server** KV cache utilization, queue depth, admission rate.
- **Pod health** — restarts, OOM events, readiness flips.
- **Network** between services and to external providers.

## 19.18 Logging Discipline

Logs are expensive and risky. Discipline:

- **Structured logs.** JSON, not free text. Searchable, parseable.
- **Trace IDs** in every log line.
- **No raw prompts or completions** unless explicitly enabled and redacted.
- **Sample** verbose logs to a fraction; do not log every token.
- **Rate-limit** error logs; one error storm should not blow up storage cost.
- **Retention policies** per log type.

## 19.19 LLM-as-Judge Evaluation

For open-ended outputs, traditional accuracy does not apply. LLM-as-judge uses a strong LLM to grade outputs against a rubric.

Best practices:
- Use a different (often stronger) model as the judge than the model under test.
- Provide a structured rubric with examples.
- Sample carefully — judging every output is expensive.
- Validate against human ratings periodically; LLM judges drift.

Frameworks: RAGAS, TruLens, Phoenix, LangSmith evals, Braintrust.

## 19.20 SLOs and Alerting

Service Level Objectives are the targets the system promises to meet. Alerts fire when SLOs are at risk.

For GenAI products, useful SLOs:
- p95 time-to-first-token below X seconds.
- p99 total response below X seconds.
- Quality eval score above X.
- Cost per query below X.
- Availability above X%.

Alert on burn rate, not raw metric values — "we are burning the latency error budget 10x faster than allowed" is more actionable than "p99 went over 5 seconds once."

## 19.21 Real-User Monitoring (RUM)

Backend metrics tell you what the server did. RUM tells you what the user experienced. For GenAI, this often diverges — slow networks, sluggish browsers, and partial renders matter.

Tools: Sentry, Datadog RUM, PostHog, custom OpenTelemetry browser SDKs.

Capture: page load time, time-to-first-interactive, time-to-first-token-rendered, errors, user actions (stop, regenerate, copy), session replay where appropriate.

## 19.22 Production Architecture

```
   Services emit OpenTelemetry signals
        |
   OTel Collector (aggregates, samples, routes)
        |
   +--- Metrics -> Prometheus / Mimir
   +--- Logs    -> Loki / OpenSearch
   +--- Traces  -> Tempo / Jaeger
   +--- LLM evals -> LangSmith / Phoenix / Langfuse
        |
   Grafana (and AI-specific UIs) for visualization
        |
   Alertmanager / PagerDuty / Opsgenie
```

## 19.23 Tradeoffs

| Choice | Win | Cost |
|---|---|---|
| Managed observability (Datadog) | Less ops | High at scale |
| Self-hosted (Prom + Grafana + Loki + Tempo) | Cheap at scale | Ops burden |
| LLM-specific (LangSmith, Phoenix) | Prompt-aware | Another vendor |
| Sampling traces | Lower cost | Miss some events |
| Storing every trace | Full visibility | Storage cost |

## 19.24 Scaling Challenges

- High cardinality metrics (per-user labels) blow up Prometheus storage.
- Log volume scales with traffic; sample aggressively.
- Trace storage scales with QPS; use tail-based sampling for full traces of important events.
- Eval cost scales with model size; sample production traffic for continuous eval.

## 19.25 Security Concerns

- Redact PII in logs, traces, and evals.
- Tenant isolation in observability — one tenant's logs should not be visible to another.
- Audit observability access; admins seeing prompts must be tracked.
- Encrypt at rest.

## 19.26 Cost Optimization

- Sample traces and logs.
- Drop low-value metrics.
- Tier old data to cold storage.
- Self-host above a cost threshold.
- Audit observability stack cost quarterly; it grows quietly.

## 19.27 Interview Questions

- Walk through the six pillars of GenAI observability.
- Why is time-to-first-token the most important latency metric?
- How do you avoid logging PII?
- Design an alerting strategy for an LLM chat product.
- Compare LangSmith, Phoenix, and Langfuse.

## 19.28 Hands-on Exercises

1. Design the dashboard for a chat product on one screen.
2. List the alerts that should page on-call for a frontier AI feature.
3. Plan the LLM-as-judge eval for a customer-facing summarizer.

## 19.29 Common Mistakes

- Logging full prompts and completions, then leaking PII.
- High-cardinality labels in Prometheus.
- Alerting on every metric, paging too often, alert fatigue.
- No continuous evaluation; quality drifts silently.
- Observability stack treated as someone else's problem.

## 19.30 Enterprise Best Practices

OpenTelemetry from day one. Three pillars plus three GenAI pillars. SLOs documented per feature. On-call rotations defined. Quarterly observability reviews. Regular fire drills. Dashboards owned and curated, not auto-generated.
