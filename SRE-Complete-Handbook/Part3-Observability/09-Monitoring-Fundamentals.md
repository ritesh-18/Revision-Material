# Chapter 9 — Monitoring Fundamentals

## 9.1 Concept Explanation

Monitoring is the practice of collecting, processing, and alerting on signals from a running system. Observability is the broader property — can you ask new questions about your system without redeploying it? Monitoring is part of observability; the larger picture includes metrics, logs, traces, events, and the ability to correlate them.

This chapter covers fundamentals. Chapters 10, 11, and 12 dive into metrics, logs, and traces specifically.

A useful framing: monitoring exists to answer three questions:
1. **Is my service healthy?** (alerting)
2. **What is my service doing right now?** (dashboards)
3. **Why did something happen?** (debugging)

Different signals serve different questions. A unified observability stack must cover all three.

## 9.2 The Three Pillars

**Metrics** are numerical measurements over time. CPU at 80%, request rate at 2000/s, latency p99 at 250ms. Compact, efficient, ideal for alerting and dashboards. Limitation: cannot answer "why."

**Logs** are textual records of events. "User 12345 logged in at 14:23:01." Rich detail, but unstructured by default. Expensive at scale.

**Traces** are records of a request's path across services. Show the timing breakdown — how long was spent in each service, where the bottleneck was. Essential in distributed systems.

These three are complementary. Metrics tell you *what* is happening. Traces tell you *where*. Logs tell you *why*.

Modern observability adds:
- **Events** — important state changes (deploys, config changes, alerts firing). Annotations for the timeline.
- **Profiles** — sampled CPU/memory profiles, useful for performance work.

## 9.3 The Four Golden Signals

Google's classic framing for what to monitor about any service:

**Latency.** Time to serve a request. Track separately for successful and failed requests — a 50ms 500 is faster than a 5s 200, but neither is good.

**Traffic.** Demand on the system. Requests per second, bytes per second, transactions per second.

**Errors.** Rate of failed requests. Includes explicit failures (5xx) and implicit ones (wrong content, slow responses).

**Saturation.** How "full" the service is. CPU usage, memory usage, queue depth, connection pool fill. Approaches 100% as you near capacity.

These four cover most monitoring needs for request-driven systems. Memorize them; they appear in every SRE interview.

## 9.4 RED Method

Tom Wilkie's variant, focused on services:

**Rate** (requests per second).
**Errors** (failed requests).
**Duration** (latency).

Simpler than the four golden signals; better for microservices where saturation is harder to measure per service.

## 9.5 USE Method

Brendan Gregg's variant, focused on resources:

**Utilization** (how busy is the resource).
**Saturation** (how much work is queued).
**Errors** (how many resource-level errors).

Apply per resource: CPU, memory, disk, network, file descriptors, sockets. Excellent for infrastructure debugging.

## 9.6 Combining RED and USE

A complete monitoring approach:
- **RED at the service level** — what users experience.
- **USE at the resource level** — what's stressed.

When RED shows latency rising, USE shows which resource is causing it. The two together pinpoint problems quickly.

## 9.7 Cardinality

**Cardinality** is the number of unique combinations of label values for a metric. `http_requests{service="api", status="200"}` has cardinality 1 per combination. Add `user_id` as a label and cardinality explodes — millions of values.

Why it matters: high-cardinality metrics blow up monitoring storage and slow queries. Many production outages of monitoring systems trace to a single high-cardinality metric.

Rules:
- Do NOT use user IDs, session IDs, request IDs as metric labels.
- Use these in logs and traces instead.
- Limit label values per metric (most rules of thumb: under 100, ideally under 20).

Some modern observability systems (Honeycomb, Lightstep) embrace high cardinality by design and price accordingly. Most teams using Prometheus suffer from accidental cardinality and learn this rule the hard way.

## 9.8 Sampling

You cannot store every event from every request at scale. Sampling reduces volume while preserving statistical fidelity.

**Head-based sampling** — decide at request start whether to record (e.g., 1 in 100). Simple, predictable, may miss interesting requests.

**Tail-based sampling** — keep all data temporarily, sample at end based on outcome (always keep errors and slow requests, sample the rest). More expensive but catches what matters.

**Reservoir sampling** — maintain a fixed-size sample that statistically represents the stream.

For traces, head sampling at 1-10% is common; tail sampling at high rates with errors-always retention is becoming the modern default.

## 9.9 Push vs Pull Metrics

**Pull (Prometheus model):** the monitoring system scrapes metrics endpoints from services every N seconds. Services expose `/metrics`; monitoring fetches.

**Push (StatsD, OpenTelemetry):** services send metrics to a collector or directly to storage.

Pull pros: monitoring system controls cadence; easy service discovery; failure to scrape is itself a signal.
Pull cons: short-lived jobs hard to scrape; firewalls between monitor and services.

Push pros: works for short-lived jobs; firewall-friendly.
Push cons: monitoring system can be overwhelmed; harder to detect missing data.

Most modern stacks support both. Prometheus is pull-first with a "Pushgateway" for batch jobs.

## 9.10 Alert Quality

Alerts come in three qualities:

**Actionable.** Tells you something is wrong AND tells you what to do (links to a runbook with steps).

**Informational.** Tells you something is happening but no immediate action needed. Goes to a chat channel, not a pager.

**Noise.** Fires on conditions that do not matter. Wastes attention.

Healthy systems have many actionable, some informational, near-zero noise. Unhealthy systems are the opposite.

Two questions for every alert:
1. **Does this require immediate human action?** If no, do not page.
2. **Is there a runbook?** If no, write one or remove the alert.

## 9.11 Symptom vs Cause Alerting

**Symptom-based alerts** fire when users are affected. "Login error rate above 5%."

**Cause-based alerts** fire on internal conditions. "Database CPU above 90%."

Both have a place. Symptom alerts catch what matters but lag (impact must be visible). Cause alerts give early warning but produce noise (a cause does not always lead to user impact).

The modern pattern: SLO-based (symptom) alerts for paging, cause alerts as tickets or dashboards. SLO burn rate alerts (Chapter 2) are symptom alerts tuned to give early warning.

## 9.12 Dashboards

Dashboards visualize the system's state for humans. A good dashboard answers a question in 5 seconds.

Hierarchy of dashboards:
- **Overview** — service health at a glance (Golden Signals).
- **Service-level** — per-service deep dive.
- **Component-level** — per database, per cache, per queue.
- **Investigation** — ad hoc queries for incidents.

Bad dashboards:
- Too many panels (information overload).
- No clear question (just collections of metrics).
- Static dashboards no one looks at.

Good dashboards:
- Tell a story (read top-to-bottom for the most important questions).
- Use consistent scales (so you can compare across time).
- Include annotations for deploys and incidents.

## 9.13 The Observability Pipeline

```
   Services emit data
        |
   Collectors (OpenTelemetry Collector, Fluent Bit, Vector, Telegraf)
        |
   Routing and sampling
        |
   Storage backends
   +-- Metrics: Prometheus, Mimir, Thanos, VictoriaMetrics, InfluxDB
   +-- Logs: Loki, Elasticsearch / OpenSearch, S3+Athena
   +-- Traces: Tempo, Jaeger, Zipkin
   +-- Profiles: Pyroscope, Parca
        |
   Query layer
        |
   Visualization (Grafana, Kibana, Datadog UI)
        |
   Alerting (Alertmanager, PagerDuty)
```

OpenTelemetry has become the de facto standard for emitting all four signal types in a vendor-neutral way.

## 9.14 Self-Monitoring

The monitoring system must monitor itself. If Prometheus is down, do you know? Three patterns:
- **Dead man's switch** — an external service expects a regular heartbeat from your monitoring. Silence triggers an external alert.
- **Cross-monitoring** — two monitoring systems watch each other.
- **Independent paging path** — pages do not depend on the same infra as the services being monitored.

A monitoring system that goes down silently is worse than no monitoring at all.

## 9.15 Real-World Use Cases

- A payment service uses RED at the service layer plus USE for the database, Redis, and Kafka. When latency rises, the team can pinpoint the culprit in 30 seconds.
- A SaaS company added burn-rate alerts on top of legacy threshold alerts. Page count dropped 80% with no missed incidents.
- A startup using a managed platform (Datadog) hit cardinality limits when a developer accidentally labeled by user ID. Bill spiked 5x. They added cardinality CI checks.

## 9.16 Tradeoffs

| Choice | Win | Cost |
|---|---|---|
| Many metrics | Comprehensive | Storage, query cost |
| Few metrics | Cheap | Blind spots |
| High-cardinality | Rich queries | Storage explosion |
| Low-cardinality | Cheap | Limited investigation |
| Aggressive sampling | Cheap | Miss rare events |
| Full retention | Complete | Expensive |
| Managed (Datadog) | Less ops | High cost at scale |
| Self-hosted Prom stack | Cheap at scale | Ops burden |

## 9.17 Scaling Challenges

- Cardinality limits hit at 10M+ active series.
- Long retention (30+ days) requires special storage (Thanos, Mimir).
- Cross-cluster federation for multi-region setups.
- Push pipelines saturate at very high event rates.

## 9.18 Security

- Logs often contain PII. Redact at ingest.
- Metrics endpoints should not be publicly exposed.
- Trace headers can leak sensitive info if not stripped.
- Multi-tenant observability requires careful per-tenant scoping.

## 9.19 Deployment Guide

Standing up observability:
1. Choose stack (open-source: Prometheus + Grafana + Loki + Tempo; managed: Datadog, Honeycomb, New Relic).
2. Deploy collector (OTel Collector recommended).
3. Instrument services (OTel SDKs).
4. Build the four service-tier dashboards.
5. Set up Golden Signals alerts.
6. Establish runbook links from alerts.

## 9.20 Monitoring Strategy (Meta)

What to monitor about your monitoring:
- Ingestion rate.
- Query latency.
- Storage usage.
- Active series / log volume.
- Alert delivery success.
- Dead man's switch heartbeat.

## 9.21 Cost Optimization

- Drop unused metrics.
- Aggregate before ingest.
- Sample logs aggressively.
- Use tiered storage (hot/warm/cold).
- Set retention by signal type.
- Audit cardinality monthly.

Observability budgets can easily reach $1M+/year at mid-scale companies. It is worth a quarterly review.

## 9.22 Interview Questions

- *Explain the four Golden Signals.*
- *Compare RED and USE methods.*
- *What is cardinality and why does it matter?*
- *How do you distinguish noise from signal in alerts?*
- *Walk me through your observability stack.*

## 9.23 Hands-on Exercises

1. For a service you know, list its four Golden Signals with specific SLI definitions.
2. Identify three high-cardinality metrics in your stack. Replace them with low-cardinality alternatives.
3. Audit your alert list. Tag each as actionable, informational, or noise. Aim for 0% noise.

## 9.24 Common Mistakes

- Labeling by user ID / request ID.
- Alerting on causes when symptoms tell the real story.
- Dashboards no one reads, kept "just in case."
- Monitoring without dead man's switch.
- Mixing logs and metrics ("just log the metric").
- No retention plan; storage grows forever.

## 9.25 Enterprise Best Practices

OpenTelemetry from day one. SLO-based paging. Strict cardinality discipline. Centralized observability platform. Quarterly cost and quality reviews. Self-monitoring with dead man's switch. Tracing across every internal service. Common dashboards templates per service tier.
