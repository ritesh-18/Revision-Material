# Chapter 10 — Metrics: Prometheus and Grafana

## 10.1 Concept Explanation

Metrics are numerical measurements collected over time. The collection is cheap and the storage is compact, which makes metrics the right primitive for alerting, dashboards, and quantitative monitoring. Every modern observability stack has a metrics layer; the dominant open-source choice is Prometheus, paired with Grafana for visualization and Alertmanager for paging.

A metric has a name (`http_requests_total`), labels (`{service="api", status="200"}`), and a value over time. Together these form a **time series**. The combination of name and labels defines uniqueness; two time series with identical name and labels are the same series.

## 10.2 Prometheus — The Dominant Stack

Prometheus is an open-source time-series database and monitoring system created at SoundCloud in 2012, now under the CNCF umbrella. It dominates because:
- Pull-based scraping makes service discovery and operations clean.
- The query language (PromQL) is powerful for time-series math.
- A vast ecosystem of integrations (exporters) covers most software.
- Self-contained — no external dependencies (no Kafka, no DB).

What Prometheus does:
- Scrape metrics from configured targets.
- Store time series in a local TSDB.
- Evaluate alerting rules.
- Push alerts to Alertmanager.

What Prometheus does NOT do:
- Long-term storage (use Thanos, Mimir, or Cortex for that).
- Push-based ingestion (use Pushgateway as a workaround).
- Highly available clustering (it is single-node by design).

## 10.3 Metric Types

Prometheus defines four primary types:

**Counter** — a monotonically increasing number. Resets only on process restart. Use for "how many things have happened?" — request count, errors, bytes written.

**Gauge** — a value that can go up or down. Use for "what is the current value?" — current memory, current queue depth, current concurrent users.

**Histogram** — a distribution of values. Buckets count how many observations fall under each threshold. Use for latency, request sizes. Enables percentile queries.

**Summary** — similar to histogram but pre-computes percentiles on the client. Less flexible but cheaper to query. Histograms are usually preferred.

The choice between histogram and summary trips people up. Use histograms unless you have a specific reason — they aggregate across instances cleanly; summaries do not.

## 10.4 PromQL

PromQL is the query language for Prometheus. A small language with enormous power.

Basic patterns (described, not coded):
- **Selector** — `metric_name{label="value"}` returns time series matching.
- **Rate** — for counters, divide change by time. Gives "events per second."
- **Aggregation** — sum, avg, max, min across labels.
- **Histograms** — quantile functions over histogram buckets.
- **Predictive** — predict future values based on trend.
- **Alerting** — combine selectors with comparison operators.

You will spend years getting fluent. Start with rate and aggregation; they cover 80% of needs.

## 10.5 Recording Rules

Recording rules pre-compute expensive queries and store the result as a new metric. Used when:
- A query is computed often (dashboards, alerts).
- The query is expensive (cross-instance aggregations, percentiles).

Example: instead of computing the p99 latency on every dashboard refresh, define a recording rule that produces `service:latency:p99` every 30s. Dashboards read the recorded metric cheaply.

Without recording rules, complex dashboards bring Prometheus to its knees.

## 10.6 Alerting Rules

Alerting rules define conditions that trigger alerts. Standard structure:
- An expression that evaluates to "for how long has this condition held?"
- A `for:` duration to suppress flapping (only fire if condition holds for N minutes).
- Labels (severity, team) for routing.
- Annotations (description, runbook URL) for the responder.

The alerting flow:
1. Prometheus evaluates rules every interval.
2. Conditions that fire become "pending" then "firing" after `for:` duration.
3. Firing alerts are sent to Alertmanager.
4. Alertmanager deduplicates, groups, routes to receivers (PagerDuty, Slack).

## 10.7 Alertmanager

Alertmanager handles the **routing** and **delivery** layer:
- **Deduplication** — many Prometheus instances may fire the same alert; Alertmanager collapses them.
- **Grouping** — multiple related alerts become one notification.
- **Routing** — different alerts go to different receivers based on labels.
- **Silencing** — suppress alerts during known issues or maintenance.
- **Inhibition** — suppress lower-severity alerts when a higher-severity one is firing.

Alertmanager is a separate process from Prometheus, deployed for HA.

## 10.8 Exporters

Exporters translate non-Prometheus metric sources to the Prometheus format. There is an exporter for nearly everything:
- **node_exporter** — Linux system metrics (CPU, memory, disk, network).
- **postgres_exporter** — PostgreSQL stats.
- **redis_exporter** — Redis stats.
- **kafka_exporter** — Kafka cluster.
- **blackbox_exporter** — external endpoint checks (HTTP, TCP, DNS, ICMP).
- **cAdvisor** — container metrics.
- **kube-state-metrics** — Kubernetes object state.

Most teams run dozens of exporters. The exporter ecosystem is one of Prometheus's strongest features.

## 10.9 Direct Instrumentation

For application services, the better path is direct instrumentation — the application exposes a `/metrics` endpoint in Prometheus format.

Libraries exist for every major language:
- Python: prometheus_client.
- Node.js: prom-client.
- Java: micrometer (via Spring Boot) or simpleclient.
- Go: prometheus/client_golang.

Standard pattern: applications include the library, define their metrics, increment them in their code, and the library handles the `/metrics` endpoint.

## 10.10 Scaling Prometheus

Single-node Prometheus runs out of room around 1-2 million active time series. For larger scale:

**Thanos** — adds long-term storage on object stores (S3, GCS), cross-cluster query, deduplication. Production-grade since ~2018.

**Cortex** / **Mimir** (Grafana) — horizontally scalable Prometheus-compatible TSDB. Built for multi-tenancy and high cardinality.

**VictoriaMetrics** — high-performance alternative, often outperforms Prometheus on the same hardware. Single binary or clustered.

**M3DB** (Uber) — large-scale Prometheus-compatible TSDB, less commonly adopted outside Uber.

For most companies, Prometheus + Thanos is the path. For very high cardinality, Mimir or VictoriaMetrics.

## 10.11 Grafana

Grafana is the de facto visualization tool for time series. It connects to Prometheus, Loki, Tempo, Elasticsearch, InfluxDB, CloudWatch, BigQuery, and dozens more.

Key concepts:
- **Data source** — connection to a backend (Prometheus, Loki, etc.).
- **Panel** — single chart in a dashboard.
- **Dashboard** — collection of panels.
- **Variable** — dropdown filters (service, environment, region).
- **Annotation** — vertical lines on charts marking events (deploys, incidents).
- **Alert** — Grafana can also produce alerts (alternative to Alertmanager).

Modern Grafana includes:
- **Explore** — ad-hoc query mode.
- **Loki / Tempo integration** — pivot from a metric spike to logs or traces.
- **Provisioning** — dashboards as code.

## 10.12 Dashboard Design

A good dashboard answers a question. Patterns:

**Service overview.** Four panels (Golden Signals), 5-second read.

**Detailed service.** RED + USE for that service's resources. 30-second read.

**Investigation.** Many panels, only used during incidents. Templates with variables for service, environment, time.

**Business.** Connect technical metrics to business outcomes (orders/minute, signups/hour).

A dashboard that does not get viewed should be deleted. Dashboards left running are tech debt.

## 10.13 Alert Design

The art of alert design is balancing sensitivity and specificity.

- **Too sensitive** — pages on noise; engineers ignore alerts; real incidents missed.
- **Too specific** — misses real issues; users find out before you do.

Patterns that produce good alerts:
- SLO burn rate alerts (Chapter 2).
- Multi-window thresholds (require condition in both short and long windows).
- Rate-based instead of count-based (5% error rate vs 100 errors per minute).
- Comparison to baseline (anomaly detection).

Anti-patterns:
- Threshold alerts on absolute counts (works at low scale, breaks at high scale).
- Alerts that fire on transient blips.
- Single-window alerts (flap on small variations).

## 10.14 Real-World Use Cases

- A microservices org runs Prometheus per Kubernetes cluster (10+ clusters), federates via Thanos, queries unified via Grafana. Each cluster's Prometheus is independent for resilience.
- An e-commerce company has a "war room" dashboard showing the entire purchase funnel: home → product → cart → checkout → payment → confirmation. Each step has a Golden Signals panel. Spots issues before customers complain.
- A SaaS company alerts on SLO burn rate only. They eliminated 90% of their old threshold alerts. MTTR improved because every page is now meaningful.

## 10.15 Production Architecture

```
   Services expose /metrics (Prometheus format)
        |
   Prometheus (per cluster) scrapes every 15-30s
        |
   Recording rules pre-aggregate
        |
   Alerting rules evaluate
        |
   Firing alerts -> Alertmanager
        |
   Alertmanager routes -> PagerDuty / Slack / email
        |
   Long-term storage -> Thanos / Mimir (S3 backend)
        |
   Grafana queries Prometheus (recent) + Thanos (older)
```

## 10.16 Alternatives to Prometheus

- **Datadog** — managed, integrated logs/metrics/traces, expensive.
- **New Relic** — managed, broad coverage, ergonomic.
- **InfluxDB** — open-source TSDB, simpler than Prometheus for some use cases.
- **OpenTelemetry Metrics** — vendor-neutral standard, increasingly the default.
- **CloudWatch** (AWS) — native, integrated with AWS services.
- **Stackdriver / Cloud Monitoring** (GCP).
- **Azure Monitor** (Azure).

Most teams pick Prometheus for self-hosted, Datadog or cloud-native for managed.

## 10.17 Tradeoffs

| Choice | Win | Cost |
|---|---|---|
| Prometheus (self-hosted) | Cheap at scale | Ops burden |
| Datadog (managed) | Less ops | High cost |
| High scrape frequency | Fine-grained | More storage |
| Low scrape frequency | Cheap | Miss spikes |
| Many labels | Rich queries | Cardinality risk |
| Few labels | Cheap | Less flexibility |

## 10.18 Scaling Challenges

- Active series count is the dominant constraint.
- Long retention requires object-store backends.
- Query performance degrades on huge series counts.
- Multi-tenancy requires Cortex/Mimir.

## 10.19 Security

- Metrics endpoints should not be public — bind to internal network or require auth.
- Alertmanager APIs (silencing) should require auth.
- Grafana role-based access control for dashboards.
- Audit logs on dashboard changes and silences.

## 10.20 Deployment Guide

A first Prometheus setup:
1. Deploy Prometheus per cluster (kube-prometheus-stack Helm chart).
2. Add node_exporter and kube-state-metrics.
3. Instrument core services.
4. Add basic recording rules.
5. Add Golden Signals alerts.
6. Deploy Alertmanager with PagerDuty integration.
7. Deploy Grafana with the kube-prometheus dashboards.
8. Add Thanos for long-term storage (when retention need exceeds local disk).

## 10.21 Monitoring Strategy (Meta)

- Prometheus active series count.
- Scrape duration / failures.
- Storage usage trend.
- Alert evaluation duration.
- Alertmanager delivery success rate.

## 10.22 Cost Optimization

- Drop unused series via metric_relabel_configs.
- Aggregate by recording rules.
- Set retention by tier (short hot, long warm/cold).
- Federate selectively.
- Audit cardinality monthly.
- Use cheaper object storage for Thanos.

## 10.23 Interview Questions

- *Explain Prometheus pull vs push.*
- *What are the four metric types?*
- *Walk me through a PromQL query for p99 latency.*
- *Compare Prometheus, Thanos, Mimir, VictoriaMetrics.*
- *How do you control cardinality?*

## 10.24 Hands-on Exercises

1. Set up Prometheus locally. Scrape node_exporter. Build a Golden Signals dashboard.
2. Write a recording rule for service-level p99 latency.
3. Write an alerting rule using multi-window burn rate.
4. Identify three high-cardinality labels in your stack. Plan removals.

## 10.25 Common Mistakes

- Using user IDs as labels.
- Histograms with too many buckets (each adds cardinality).
- Querying raw data when a recording rule would do.
- Long retention on local Prometheus disk.
- No HA Alertmanager — single point of paging failure.
- Mixing logs and metrics ("log a counter").

## 10.26 Enterprise Best Practices

Standardize the metrics stack across teams. Use kube-prometheus-stack as baseline. Federate via Thanos or migrate to Mimir. Make recording rules mandatory for dashboard queries. Cardinality reviews quarterly. Provision dashboards as code (git-backed). Train teams in PromQL.
