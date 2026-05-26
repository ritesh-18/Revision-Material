# Chapter 36 — SRE System Design

## 36.1 Concept Explanation

SRE system design is the practice of designing systems with reliability, observability, scalability, and cost as first-class concerns. It overlaps with general software architecture but emphasizes operational properties: how does this behave under load, under failure, under attack, over time?

In interviews and reviews, SRE system design questions ask: "Design a system that is reliable at scale." The answer is rarely the same as "design a system that works." Production properties demand specific patterns.

## 36.2 The Method

A reliable SRE system design process:

1. **Requirements.** Functional + non-functional (SLOs, throughput, latency, cost envelope).
2. **High-level design.** Boxes and arrows.
3. **Component design.** Stateless services, data stores, queues.
4. **Failure modes.** What breaks; how do we know; what do we do.
5. **Scaling.** What changes at 10x, 100x, 1000x.
6. **Observability.** What is measured.
7. **Security.** Identity, access, encryption.
8. **Cost.** Order-of-magnitude estimate.
9. **Operations.** How does on-call deal with this.

This list looks like the SRE-flavored version of any system design framework. The emphasis matters: reliability and operations get top billing.

## 36.3 Design 1 — A Highly Available Web Service

**Requirements.** 99.95% availability, p99 latency < 500ms, 10M MAU.

**HLD.**
```
   Users
     |
   CDN (Cloudflare)
     |
   Multi-region DNS with health checks
     |
   ALB per region
     |
   Stateless app tier (EKS, autoscaled)
     |
   +-- Aurora primary (multi-AZ) + read replicas
   +-- ElastiCache Redis
   +-- S3 for objects
     |
   Observability mesh (Prometheus, Loki, Tempo)
```

**Reliability.**
- Multi-AZ for everything.
- Read replicas for read scaling.
- Auto-scaling on RPS.
- Health checks driving routing.

**Failure modes.**
- AZ outage → other AZs absorb.
- Region outage → DR plan (active-passive cross-region).
- DB primary failure → automatic Aurora failover.
- Surge → autoscaling + rate limiting.

**Observability.** RED per service, USE per node, business metrics, SLO burn alerts.

**Cost.** Reserved instances for baseline, on-demand for surge.

## 36.4 Design 2 — A Real-Time Notification System

**Requirements.** Deliver notifications to mobile clients globally. 100M+ devices. <2s end-to-end p99.

**HLD.**
```
   Notification API
     |
   Kafka (partitioned by user)
     |
   Worker pool (consume by partition)
     |
   Push provider integrations (APNS, FCM, web push)
     |
   Retry queue for failures
     |
   Dead-letter for terminal failures
     |
   Per-user state in Redis / DynamoDB
```

**Reliability.**
- Kafka at-least-once semantics; idempotent consumers.
- Per-partition workers for ordering per user.
- DLQ for unrecoverable failures.
- Multi-region with regional Kafka clusters.

**Scaling.** Partitions by user ID; add partitions and workers proportionally.

**Cost.** Most cost is Kafka and worker compute; right-size workers.

## 36.5 Design 3 — A Multi-Tenant SaaS Platform

**Requirements.** 1000 enterprise tenants, strict isolation, SOC 2, GDPR.

**HLD.**
```
   Tenant authentication (SSO per tenant)
     |
   API gateway with tenant routing
     |
   App tier (shared, tenant-aware)
     |
   +-- Per-tenant database schemas (or per-tenant DBs)
   +-- Per-tenant object store prefixes
   +-- Per-tenant rate limits
   +-- Per-tenant audit logs
     |
   Cross-region for residency
     |
   Compliance evidence collection
```

**Isolation.**
- Per-tenant schemas (logical) or per-tenant DBs (physical) by data sensitivity.
- Per-tenant encryption keys for the most sensitive customers.
- Audit log per tenant action.
- Rate limit per tenant to prevent noisy neighbor.

**Compliance.**
- Data residency per tenant.
- Right-to-delete pipeline.
- Audit log retention.

## 36.6 Design 4 — Globally Distributed Search

**Requirements.** Sub-200ms p99 for queries from any continent. Billion-document index.

**HLD.**
```
   User
     |
   Anycast edge
     |
   Edge LLM or rule routing
     |
   Per-region search index (Elasticsearch or specialized)
     |
   Per-region cache
     |
   Origin source of truth (cross-region replicated)
```

**Reliability.**
- Per-region indices to avoid cross-region calls.
- Anycast for routing to nearest region.
- Async index updates from origin.

**Tradeoffs.** Index freshness lags by replication delay; usually acceptable for search.

## 36.7 Design 5 — A Payment Processing System

**Requirements.** Process credit card payments. 100K TPS peak. Zero data loss. PCI-DSS.

**HLD.**
```
   Customer site
     |
   Payment widget (PCI scope contained)
     |
   Tokenization service (replaces card with token)
     |
   Payment orchestrator
     |
   +-- Card network integrations
   +-- Risk scoring service
   +-- Fraud detection
   +-- Settlement queue
     |
   Audit log immutable
     |
   Reporting database (read-only replica)
```

**Reliability.**
- Idempotency keys per transaction.
- Synchronous response with async settlement.
- Multi-region active-active.
- Strict consistency on financial state.

**Security.**
- PCI segmentation.
- Encryption at rest with HSM-backed keys.
- Audit logs immutable.
- Strict RBAC.

## 36.8 Design 6 — An AI Chatbot Platform

**Requirements.** Serve LLM-based chat to millions of users. p95 TTFT < 1.5s. Cost under control.

**HLD.**
```
   Users
     |
   Edge / CDN
     |
   API gateway with auth + rate limits
     |
   LLM gateway (LiteLLM or custom)
   +-- Model routing (small vs large)
   +-- Prefix cache
   +-- Semantic cache
   +-- Provider failover
     |
   +-- Self-hosted vLLM clusters (H100, A10G)
   +-- Provider APIs (Anthropic, OpenAI)
     |
   Per-user token budgets
     |
   RAG pipeline (vector DB + reranker) for retrieval
     |
   Observability with eval pipeline
```

**Cost.**
- Cache aggressively.
- Route by difficulty.
- Self-host where break-even.
- Per-user caps.

**Quality.**
- Eval pipeline catching regressions.
- Safety filters.

## 36.9 Design 7 — A Batch ML Training Pipeline

**Requirements.** Train models daily on large datasets. Reliable, observable, cost-aware.

**HLD.**
```
   Data sources
     |
   Ingest pipeline (Airflow / Dagster)
     |
   Data lake (Iceberg on S3)
     |
   Feature engineering
     |
   Training job (GPU cluster, spot capacity)
     |
   Model artifact store (MLflow)
     |
   Eval pipeline
     |
   Promotion to inference (versioned)
```

**Reliability.**
- Checkpointing for long training runs.
- Spot interruption handling.
- Retry policies.

**Cost.**
- Spot GPUs save 70%.
- Right-size cluster.
- Schedule during off-peak.

## 36.10 Design 8 — A Real-Time Analytics Dashboard

**Requirements.** Live dashboards updating every 5 seconds. Aggregations over billions of events. <2s query latency.

**HLD.**
```
   Event sources
     |
   Kafka
     |
   Stream processor (Flink, ksqlDB, Materialize)
     |
   Aggregated state in fast store (Druid, ClickHouse, Pinot)
     |
   API serving dashboards
     |
   Frontend (React with WebSocket for live updates)
```

**Reliability.**
- Kafka durability.
- Stream processor checkpointing.
- Idempotent aggregations.

**Performance.**
- Pre-aggregation in stream processor.
- Columnar query engine.

## 36.11 Cross-Cutting Concerns

Across all designs:

**Observability.** RED + USE everywhere. Trace IDs propagated. SLOs documented.

**Security.** Auth at edge. IAM least privilege. Encryption at rest + transit. Audit logs.

**Cost.** Tagged for attribution. Reviewed quarterly. Budgets enforced.

**Operations.** Runbooks. On-call. Postmortems. DR plans.

These are not optional; they appear in every senior design.

## 36.12 The Interview

In an SRE system design interview:

1. **Clarify** requirements aggressively. Latency, throughput, availability, region scope, compliance, cost.
2. **High-level diagram** within 10 minutes.
3. **Identify hot paths and stateful components.**
4. **Discuss reliability for each.** Failure modes; recovery.
5. **Discuss scaling.** What changes at 10x.
6. **Observability.** What you measure.
7. **Estimate cost.** Order of magnitude.
8. **Address the interviewer's pushes.** Don't dig in defensively.

The interviewer is testing thinking, not memorization. Trade off out loud.

## 36.13 Tradeoffs Are the Answer

Every choice in design is a tradeoff. Naming them is mature SRE practice:

- Consistency vs availability vs latency (PACELC).
- Cost vs reliability vs performance.
- Simple vs flexible.
- Generic vs purpose-built.
- Build vs buy.
- Centralized vs distributed.

Senior SREs do not say "I would use X." They say "X has these tradeoffs vs Y; given our constraints, X wins because Z."

## 36.14 Real-World Use Cases

System designs you should be able to discuss:
- Twitter feed.
- WhatsApp messaging.
- Uber dispatching.
- Netflix streaming.
- Google search.
- ChatGPT-like chat.
- Stripe payments.
- Slack messaging.

Read postmortems and engineering blogs from these companies. Internalize the patterns.

## 36.15 Tradeoffs Cheat Sheet

| Pattern | Pros | Cons |
|---|---|---|
| Synchronous | Simple flow | Tight coupling |
| Async via queues | Decoupled, resilient | Eventual, debug harder |
| Single region | Cheap, simple | Region risk |
| Multi-region | Resilient | Complex, expensive |
| Strong consistency | Predictable | Latency, availability |
| Eventual consistency | Available, fast | Anomalies |
| Monolith | Simple ops | Coupled scaling |
| Microservices | Independent scaling | Operational overhead |
| Push | Real-time | More state |
| Pull | Simple | Polling overhead |

## 36.16 Interview Questions

- *Design [X]* — pick one of the eight above and walk through 5-15 minutes.
- *How does this scale to 10x users?*
- *What if your primary database goes down?*
- *Where does state live, and why?*
- *How do you observe this in production?*

## 36.17 Hands-on Exercises

1. Pick a product. Design the SRE-focused architecture in one page.
2. For one component, list all failure modes and mitigations.
3. Calculate the rough cost at 100K users and 10M users.

## 36.18 Common Mistakes

- Jumping to tools before requirements.
- Single-point-of-failure designs.
- No observability mentioned.
- No cost discussion.
- "Just use X" without explaining tradeoffs.

## 36.19 Enterprise Best Practices

Design reviews include reliability and operations. Patterns documented (golden paths). Production readiness gates enforce reliability. Cost estimates required before major investments. Senior engineers review designs for failure modes. Postmortem patterns feed back into design guidelines.
