# Chapter 11 — Logs: Loki and the ELK Stack

## 11.1 Concept Explanation

A log is a record of an event. "User 1234 logged in at 14:32:01 from IP 1.2.3.4." Logs are the original observability signal — every UNIX program writes them. The difficulty is not generating logs; it is finding the ones you need at the moment you need them, at the scale modern systems produce them.

A medium-size SaaS company generates 100 GB to 10 TB of logs per day. A large one generates petabytes. Naively storing and searching all of it is expensive. Modern logging stacks index intelligently, tier storage, and let you query the haystack quickly.

## 11.2 The Logging Spectrum

Logs serve different purposes at different layers:

**Application logs** — what your code wrote. Business events, errors, debug info.

**Infrastructure logs** — what the OS, runtime, and platform wrote. systemd, kernel, container runtime, Kubernetes events.

**Audit logs** — security and compliance events. Logins, permission grants, config changes. Often required to be tamper-evident.

**Access logs** — HTTP request logs from gateways and load balancers.

**Database logs** — slow queries, errors, replication events.

Each layer has different volume, retention, and access patterns. Treating them all the same wastes money or loses important data.

## 11.3 Structured vs Unstructured Logs

**Unstructured** — free-form text. "User 1234 logged in." Easy to write, hard to query.

**Structured** — fields with values, usually JSON. `{"user_id": 1234, "event": "login", "ip": "1.2.3.4", "timestamp": "..."}` Easy to query, larger to store.

Modern practice: structured logs everywhere. The cost per byte is trivial; the operational value is enormous.

A common convention: every log line is a JSON object with at least `timestamp`, `level`, `message`, `service`, `trace_id`. Additional fields per event type.

## 11.4 Log Levels

The standard hierarchy:
- **FATAL / CRITICAL** — about to terminate.
- **ERROR** — something failed.
- **WARN** — something unusual but not failure.
- **INFO** — significant business events.
- **DEBUG** — detail useful only when investigating.
- **TRACE** — even more detail, usually off.

Production usually runs at INFO. ERROR triggers alerting. DEBUG can be enabled temporarily for incident investigation, but always-on DEBUG is expensive.

## 11.5 The ELK Stack

Elasticsearch + Logstash + Kibana, with Beats (lightweight forwarders) increasingly added. The classic logging stack from the 2010s.

**Elasticsearch** — distributed search engine, the storage and query layer.
**Logstash** — ingestion pipeline with parsers and enrichers.
**Kibana** — visualization and search UI.
**Beats** — lightweight log shippers (Filebeat for files, Metricbeat for metrics).

Strengths: powerful full-text search, mature, broad community, rich aggregations.
Weaknesses: expensive at scale (every field is indexed), operationally heavy (JVM tuning, cluster management), per-document storage cost high.

**OpenSearch** is the Amazon/Apache fork after the Elasticsearch license change. Functionally equivalent for most uses.

## 11.6 Loki

Loki, from Grafana Labs, is "Prometheus for logs." It indexes only metadata (labels) and stores log content as compressed chunks in object storage. This makes it dramatically cheaper than Elasticsearch.

Tradeoffs:
- Loki cannot do full-text search efficiently the way Elasticsearch can.
- Loki excels when you can filter by labels (service, environment, region) and then grep within the matching streams.
- Loki integrates natively with Prometheus and Grafana — pivot from metrics to logs effortlessly.

For most teams, Loki is now the better default. Pay Elasticsearch prices only when you genuinely need its search features.

## 11.7 Other Logging Backends

- **Cloud-native** — CloudWatch Logs (AWS), Cloud Logging (GCP), Azure Monitor.
- **Datadog Logs** — integrated with Datadog observability.
- **Splunk** — the enterprise giant. Powerful, expensive.
- **Sumo Logic** — managed alternative.
- **ClickHouse / VictoriaLogs** — column-store backends for very high volume.
- **S3 + Athena** — for archive-only retention with occasional query.

Many large companies use a tiered architecture: Loki for hot recent data, S3+Athena for cold archives, Splunk for security/compliance.

## 11.8 The Logging Pipeline

```
   Application writes logs (stdout or file)
        |
   Forwarder (Fluent Bit, Vector, Filebeat, Promtail)
        |
   Aggregator / processor (Fluentd, Logstash, Vector)
        |
   Routing (by tenant, by severity, by type)
        |
   Storage (Loki, Elasticsearch, S3)
        |
   Query layer (Grafana, Kibana, Athena)
        |
   Alerting (Loki rules, Elasticsearch watchers)
```

**Fluent Bit** is the modern forwarder of choice — small, fast, written in C.
**Vector** (Datadog) is a powerful alternative supporting metrics, logs, and traces in one pipeline.

## 11.9 Indexing Strategies

The cost-feature tradeoff in log systems:

**Index everything** (Elasticsearch default). Fast search on any field. Expensive storage.

**Index labels, not content** (Loki). Cheap storage. Fast filter by label, slow text grep.

**No indexing** (S3 + Athena). Cheapest storage. Slow ad-hoc query.

Most teams want hot data indexed (Loki/Elasticsearch), cold data unindexed (S3).

## 11.10 Sampling

At very high log volume, sampling reduces cost. Approaches:

**Head sampling** — decide at log time whether to keep (e.g., 10% of INFO, 100% of ERROR).

**Tail sampling** — buffer, then keep based on outcome (always keep error context).

**Dynamic sampling** — adjust rate based on traffic patterns.

Logs that get sampled out are gone forever. Be careful what you sample.

## 11.11 PII and Redaction

Logs are notorious for accidental PII inclusion. Credit card numbers, emails, passwords, JWT tokens all leak into logs regularly.

Defenses:
- **At source** — applications avoid logging sensitive fields.
- **At pipeline** — Fluent Bit/Vector strip known sensitive patterns.
- **At storage** — encryption at rest.
- **At access** — RBAC on log queries.

Modern compliance (GDPR, HIPAA) demands robust handling. Plan for it.

## 11.12 Log Retention

Retention is the dominant cost variable. Patterns:

- **Hot tier (Loki/Elasticsearch)** — 7-30 days. Fast query.
- **Warm tier (cheaper storage)** — 30-90 days. Slower query.
- **Cold tier (S3 Glacier)** — months to years. Rare query.
- **Audit logs** — separate retention, often 7 years for compliance.

Setting tiers right reduces logging costs by 5-10x.

## 11.13 Searching Logs Effectively

Three patterns:

**Filter first, then search.** Narrow by labels (service, environment, time range) before grepping text. Loki excels here.

**Index everything important.** If you regularly search by `user_id`, make it a label or indexed field.

**Pre-aggregate common queries.** Convert "count of errors per service per minute" into a metric, query as metric.

Bad search strategy is the #1 cause of "logging is too expensive."

## 11.14 Log-Based Alerting

You can alert on logs:
- Loki alert rules count matching log lines.
- Elasticsearch Watcher or alerting plugins do the same.
- Convert log patterns to metrics (via log_to_metric pipelines), then alert on metrics.

The last approach is the modern preference — metric-based alerts are cheaper and faster.

## 11.15 Trace IDs in Logs

Every log line should include the trace ID. This is what lets you pivot from one signal to another:

- See a slow request in a trace → look up logs by trace ID.
- See an error in a log → look up the full trace.

OpenTelemetry standardizes the propagation of trace IDs across services. Logging libraries pick them up from context.

## 11.16 Real-World Use Cases

- A team's Elasticsearch cluster cost $200k/month. Migrated to Loki: $30k/month, with 10% slower search. Worth it.
- A company's audit logs were intermingled with application logs. After a compliance audit, they split them — audit logs to a tamper-evident store with separate retention.
- A startup discovered they were logging full request bodies including credit card numbers. Fixed at the SDK level; ran a 30-day cleanup on old logs.

## 11.17 Production Architecture

```
   Application stdout logs (JSON)
        |
   Container runtime captures
        |
   Fluent Bit / Promtail (DaemonSet on K8s nodes)
        |
   Aggregator (per region, optional)
        |
   Routing (by label) to:
   +-- Loki (hot, 14 days)
   +-- S3 (cold, 1 year)
   +-- Splunk (security/audit subset, 7 years)
        |
   Grafana for query
        |
   Loki rules for alerting on patterns
```

## 11.18 Alternatives Comparison

| Stack | Cost | Search Power | Ops Effort |
|---|---|---|---|
| Elasticsearch / OpenSearch | High | High | High |
| Loki | Low | Medium (label-filtered) | Medium |
| Splunk | Very high | Very high | Medium (managed) |
| Datadog Logs | High | High | Low (managed) |
| ClickHouse | Low | Medium-high | Medium-high |
| S3 + Athena | Lowest | Low (ad hoc only) | Low |

Choose by your access pattern: frequent full-text search → Elasticsearch; mostly label-filtered with grep → Loki; archive with rare query → S3+Athena.

## 11.19 Tradeoffs

| Choice | Win | Cost |
|---|---|---|
| Index everything | Fast search | Expensive |
| Label-only indexing | Cheap | Slower search |
| Long retention | Available history | Storage cost |
| Short retention | Cheap | Lost data |
| Full-text indexing | Powerful | Index size |
| Sampling | Cheap | Lost events |

## 11.20 Scaling Challenges

- Log volume often grows faster than traffic.
- Cardinality explosion in labels (similar to metrics).
- Multi-region replication of logs.
- Compliance audits need consistency.

## 11.21 Security

- Encryption at rest and in transit.
- RBAC on log access.
- PII detection and redaction at pipeline.
- Audit logging on access to sensitive logs.
- Tamper-evident storage for audit logs (immutable buckets, append-only).

## 11.22 Deployment Guide

Standing up logging:
1. Decide structured format (JSON with required fields).
2. Deploy collector (Fluent Bit as DaemonSet on K8s).
3. Choose backend (start with Loki).
4. Set retention (default 14 days hot, 90 days S3).
5. Configure PII redaction.
6. Add trace IDs to logs.
7. Build standard dashboards (errors per service, slow queries, security events).
8. Test query performance under realistic load.

## 11.23 Monitoring Strategy (Meta)

- Log ingestion rate.
- Storage usage trend.
- Query latency.
- Pipeline errors (dropped logs).
- Failed PII redactions.

## 11.24 Cost Optimization

- Drop noisy DEBUG/INFO at the source for stable services.
- Aggregate before storing.
- Tier hot/warm/cold.
- Use Loki where Elasticsearch isn't needed.
- Audit per-service log volume monthly.

## 11.25 Interview Questions

- *Compare Elasticsearch and Loki.*
- *How do you handle PII in logs?*
- *Describe a logging pipeline end to end.*
- *When would you alert on logs vs metrics?*
- *How do you scale a logging stack?*

## 11.26 Hands-on Exercises

1. Set up Loki + Promtail + Grafana locally. Tail container logs.
2. Add trace IDs to your application logs.
3. Audit your logs for PII. List three fields to redact.
4. Calculate yearly cost for your current log volume in Loki vs Elasticsearch vs S3.

## 11.27 Common Mistakes

- Unstructured logs (free-text strings).
- No trace IDs.
- Logging full HTTP bodies (PII leak).
- Long retention on hot storage.
- Single tier for all log types.
- DEBUG always on in production.

## 11.28 Enterprise Best Practices

Structured JSON everywhere. Trace IDs in every log. PII redaction at pipeline. Tiered storage with explicit retention per tier. Per-service log budgets. Quarterly review of log volume and cost. Audit logs separated and tamper-evident.
