# Chapter 12 — Distributed Tracing

## 12.1 Concept Explanation

A trace is the record of a request's path through a distributed system. Each step the request takes — through services, databases, caches, queues — is a **span**. Spans link to their parent (the span that called them), forming a tree. The tree represents one request end to end.

Traces answer questions metrics and logs cannot:
- Where in the call graph did time go?
- Which dependency was slow?
- What sequence of calls did this request actually make?
- Why is this user's request slow when the average is fast?

In monolithic systems, a stack trace gives this answer. In distributed systems, the stack trace ends at the network boundary; tracing fills the gap.

## 12.2 Spans and Traces

A **span** has:
- An ID (unique per span).
- A trace ID (shared across all spans in one trace).
- A parent span ID (for the tree structure).
- A name (operation: `db.query`, `http.get`, `cache.lookup`).
- A start time and duration.
- Attributes (key-value pairs: `http.status`, `db.statement`, `user.id`).
- Events (timestamped logs within the span).
- Status (ok, error).

A **trace** is the collection of all spans sharing one trace ID. Visualized as a "flame chart" where time goes left to right and depth shows the call hierarchy.

## 12.3 Context Propagation

For traces to work, the trace ID must flow with the request across services. This is **context propagation**.

The W3C standard uses two HTTP headers:
- `traceparent` — carries the trace ID, parent span ID, and flags.
- `tracestate` — vendor-specific extensions.

Earlier formats (B3 from Zipkin, X-Trace from custom systems) are still around. OpenTelemetry handles all of them.

Without correct propagation, traces break at service boundaries — you see local stacks but cannot connect them.

## 12.4 OpenTelemetry — The Standard

OpenTelemetry (OTel) is the unified standard for instrumentation across metrics, logs, and traces. It is the merger of OpenTracing and OpenCensus projects, governed by the CNCF.

What OTel provides:
- **SDKs** for every major language to instrument applications.
- **API** for declaring spans and metrics.
- **Auto-instrumentation** libraries that wire up common frameworks (HTTP servers, ORMs, gRPC clients) automatically.
- **Collector** — a deployable component that receives, processes, and exports telemetry.
- **Wire protocol** (OTLP) that any backend can ingest.

Adopt OTel from day one. The vendor neutrality is worth it — you can swap backends without re-instrumenting.

## 12.5 The OpenTelemetry Collector

The Collector is a separately deployed binary that:
- **Receives** telemetry from OTel SDKs.
- **Processes** it (sampling, attribute filtering, batching).
- **Exports** to one or more backends (Jaeger, Tempo, Datadog, Honeycomb).

Why use a Collector? Decoupling. Applications send to the Collector; the Collector decides where it goes. Change backends without redeploying apps.

The Collector also batches and samples, reducing load on backends and on the network.

## 12.6 Tracing Backends

**Jaeger** — open-source, originally from Uber, now CNCF. Mature, broadly used. Storage in Cassandra, Elasticsearch, or its own KV store.

**Tempo** (Grafana Labs) — modern, uses object storage (S3) for cost efficiency. Pairs naturally with Loki and Prometheus in the Grafana stack. Becoming the open-source default.

**Zipkin** — older, simpler. Still used in some shops but Tempo and Jaeger dominate new deployments.

**Datadog APM, Honeycomb, Lightstep, New Relic, Splunk Observability Cloud** — managed.

**X-Ray** (AWS) — native, integrated with AWS services.

For most modern deployments: OTel Collector + Tempo, or OTel + a managed backend.

## 12.7 Sampling Strategies

You cannot store every span from every request — the cost is prohibitive at scale. Sampling reduces volume.

**Head sampling** — decide at trace start whether to record. Fixed rate (e.g., 1%) or probabilistic.

Pros: predictable cost, simple.
Cons: random — interesting traces (slow, error) get dropped at the same rate as boring ones.

**Tail sampling** — collect all spans temporarily, decide at trace end based on outcome. Keep all errors, all slow traces, sample the rest at a low rate.

Pros: catches what matters.
Cons: requires buffering, more complex, the Collector becomes stateful.

**Adaptive sampling** — adjust rates based on service load.

Most modern setups use tail-based sampling with hard-keep rules for errors and slow traces, then 1-5% sampling for the rest.

## 12.8 What to Instrument

Auto-instrumentation handles most of this; manual instrumentation fills gaps.

**Auto-instrumented (usually):**
- HTTP servers (incoming requests).
- HTTP clients (outgoing requests).
- gRPC clients and servers.
- Database calls (most ORMs).
- Cache calls (Redis, Memcached).
- Message brokers (Kafka, RabbitMQ).
- Cloud SDK calls.

**Manual instrumentation needed for:**
- Business operations (a span for "process_order").
- Long internal computations.
- Custom protocols.
- Critical decision points.

## 12.9 Span Attributes

Attributes add searchable context to spans. Standard OpenTelemetry attribute names:
- `http.method`, `http.status_code`, `http.url`.
- `db.system`, `db.statement`, `db.name`.
- `messaging.system`, `messaging.destination`.
- `peer.service`, `net.peer.host`.

Plus your own: `user.id`, `tenant.id`, `request.size`, `feature.flag`. (Be careful with cardinality and PII.)

Attributes are why traces are powerful — you can ask "show me slow traces where user.id = 1234" rather than scanning all traces.

## 12.10 Errors in Traces

A span has a status (ok, error). When a span has error status:
- Visible in trace UIs.
- Queryable separately.
- Often auto-sampled even with head sampling.
- Used for error rate calculations.

Manual or auto-instrumentation must set the error status — it does not automatically infer from HTTP 500.

## 12.11 Visualization

Trace UIs typically show:
- **Flame chart / Gantt chart** — time on x-axis, span hierarchy on y-axis. The default view.
- **Service map** — which services called which during this trace.
- **Latency breakdown** — time spent per service.
- **Logs in context** — logs from this trace's services during this trace's time range.

Grafana Tempo + Loki + Prometheus offers powerful pivots — click a span, see logs from that service at that time, click logs to see related metrics.

## 12.12 RED at the Trace Level

Traces enable per-operation RED metrics:
- Per-endpoint latency distributions.
- Per-database query latency.
- Per-dependency error rate.

These are richer than metrics alone because they reflect actual end-to-end paths, not synthetic checks.

## 12.13 Service Dependency Graph

Aggregating traces over time produces a service map: which services call which, how often, with what latency and error rate.

Service maps are invaluable for:
- Architecture documentation.
- Identifying unexpected dependencies.
- Capacity planning.
- Migration planning.

## 12.14 Real-World Use Cases

- A team's checkout flow was slow. Trace analysis showed 80% of the time was in a single recommendation call. Disabled the call during checkout; latency dropped 4x.
- A debugging session: "Why does this user's request take 8 seconds?" Trace revealed a database query that did a full table scan due to a missing index for that user's data shape. Index added; query dropped to 50ms.
- A migration: replacing a legacy service. Traces showed every consumer of the old service. Could not have done a clean migration without that data.

## 12.15 Production Architecture

```
   Applications instrumented with OTel SDK
        |
   Send via OTLP to local Collector
        |
   Collector batches and samples
        |
   Exports to:
   +-- Tempo (storage)
   +-- Optional: managed backend (Datadog, Honeycomb)
   +-- Metric exporters (extract RED metrics from spans)
        |
   Grafana queries Tempo
        |
   Linked to Loki (logs) and Prometheus (metrics)
   for unified pivoting
```

## 12.16 Tradeoffs

| Choice | Win | Cost |
|---|---|---|
| Head sampling | Predictable, simple | May lose interesting traces |
| Tail sampling | Keeps what matters | Complex, stateful Collector |
| 100% retention | Complete | Very expensive |
| Auto-instrumentation only | Easy | May miss business operations |
| Manual instrumentation | Rich detail | Engineering time |
| Managed backend | Less ops | High cost |
| Self-hosted Tempo | Cheap | Operations burden |

## 12.17 Scaling Challenges

- Span volume scales with traffic × services × samples. Grows fast.
- Collector becomes a single point that must scale.
- Tail sampling requires holding all spans of a trace until completion.
- Cross-cluster traces need consistent collector configuration.

## 12.18 Security

- Trace attributes can include sensitive data (PII, tokens). Strip at Collector.
- Trace IDs in logs are fine, but full traces shouldn't be accessible to everyone.
- RBAC on trace queries.
- Audit access to traces of regulated transactions.

## 12.19 Deployment Guide

A staged rollout:
1. Deploy OTel Collector (one per cluster or region).
2. Instrument one critical service with auto-instrumentation.
3. Verify traces appear in backend.
4. Add manual spans for business operations.
5. Roll out to remaining services.
6. Configure tail sampling.
7. Build standard dashboards (RED per service from spans).
8. Train teams in trace investigation.

## 12.20 Monitoring Strategy (Meta)

- Spans ingested per second.
- Collector CPU/memory.
- Sampling rate and dropped spans.
- Backend query latency.
- Trace storage usage.

## 12.21 Cost Optimization

- Sample aggressively where appropriate (most traces are boring).
- Use object storage backends (Tempo, not Cassandra).
- Strip large attributes (don't store full HTTP bodies in spans).
- Tier hot/warm storage if backend supports it.
- Audit per-service span count quarterly.

## 12.22 Interview Questions

- *What is a span vs a trace?*
- *How does context propagation work?*
- *Compare head sampling and tail sampling.*
- *Walk me through using traces to debug a latency issue.*
- *What is OpenTelemetry and why does it matter?*

## 12.23 Hands-on Exercises

1. Set up OTel + Tempo + Grafana locally. Instrument a small app. View traces.
2. Add manual spans to a service for business operations.
3. Configure tail-based sampling rules.
4. Use traces to investigate one slow endpoint in a system you know.

## 12.24 Common Mistakes

- No context propagation — traces break at service boundaries.
- Sampling errors and slow traces at the same rate as the rest.
- Storing HTTP bodies in span attributes.
- High-cardinality span names (e.g., `GET /users/1234` vs `GET /users/:id`).
- No business operations spans (only HTTP-level visibility).
- Vendor lock-in without OpenTelemetry adoption.

## 12.25 Enterprise Best Practices

OpenTelemetry as the wire standard. OTel Collectors deployed per region. Tail-based sampling with error/slow-request hard keeps. Service map maintained automatically. Pivot from metrics to traces to logs by trace ID built into dashboards. Standard span attributes documented organization-wide. Quarterly storage cost reviews.
