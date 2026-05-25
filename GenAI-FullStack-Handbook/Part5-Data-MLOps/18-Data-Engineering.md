# Chapter 18 — Data Engineering for GenAI

## 18.1 Concept Explanation

GenAI applications consume and produce data at scale. Documents flow in, embeddings flow out, conversations are logged, evaluations are recorded, drift metrics are computed. Underneath every shipped feature is a data pipeline.

Data engineering for GenAI shares fundamentals with classical data engineering — ETL, streaming, warehouses, governance — but the unit of work shifts toward unstructured text, vectors, and event streams of user interactions. This chapter covers the patterns and tools that meet GenAI's specific needs.

## 18.2 ETL vs ELT

**ETL (Extract, Transform, Load).** Pull data, transform in a separate compute layer, load the transformed data into the destination. Classical pattern; still common for sensitive transformations.

**ELT (Extract, Load, Transform).** Pull data, load raw into the warehouse, transform inside the warehouse. Modern default — warehouses are powerful and store is cheaper than compute.

For GenAI: source documents are usually ingested raw, then transformed (parsed, chunked, embedded) as a separate stage. The transformation is so different from typical SQL that the boundary between ETL and ELT blurs. The principle remains: keep raw source data preserved, compute downstream artifacts from it.

## 18.3 Streaming vs Batch

**Batch.** Data processed on a schedule (nightly, hourly). Simple, cheap, latency in minutes-to-days.

**Streaming.** Data processed as it arrives. Higher complexity, latency in milliseconds-to-seconds.

GenAI ingestion is often batch (re-embed corpus nightly). User interaction logging is streaming (every chat message recorded). Many systems run both — batch for warehousing and analytics, streaming for real-time features and observability.

## 18.4 Apache Kafka

Kafka is the dominant distributed event streaming platform. Producers write events to topics; consumers read at their own pace; topics are partitioned and replicated for scale and durability.

For GenAI:
- Stream user interactions (chat messages, ratings, tool calls).
- Stream model outputs for downstream eval and analytics.
- Decouple ingestion services from embedding workers (producer writes new documents, embedding worker consumes and processes).
- Feed observability and audit logs.

Strengths: durable, ordered, replayable, battle-tested at scale. Weaknesses: operationally heavy, requires Zookeeper or KRaft mode, JVM-based.

## 18.5 Redpanda

Redpanda is a Kafka-compatible streaming platform written in C++. Drop-in replacement at the API level, much lighter operationally, often higher throughput per node.

When to choose: new deployments where Kafka compatibility is important but ops cost is high. Existing Kafka deployments rarely migrate.

## 18.6 NATS, RabbitMQ, Redis Streams

Alternatives for lighter-weight messaging:

- **NATS / NATS JetStream.** Lightweight, high-performance pub/sub with optional persistence. Strong for service-to-service eventing.
- **RabbitMQ.** Mature message broker, strong on rich routing and queue semantics.
- **Redis Streams.** Streaming on top of Redis, simple for small-to-medium scale.

For GenAI, RabbitMQ and NATS often handle task queues (embedding jobs, agent steps), while Kafka handles event logs.

## 18.7 Event-Driven Architecture

In an event-driven architecture, services emit events when state changes; other services react. Loose coupling, scalability, easier evolution.

For GenAI:
- New document arrives → emit "document_ingested" → embedding worker reacts.
- User submits feedback → emit "feedback_received" → eval pipeline updates.
- Agent completes task → emit "task_completed" → analytics ingest.

The discipline: well-defined event schemas, idempotent consumers, dead-letter queues for failures, schema registry for evolution.

## 18.8 Change Data Capture (CDC)

CDC captures row-level changes from a database and streams them as events. Useful for keeping vector indices in sync with source systems — when a document table changes, emit a CDC event and re-embed the affected document.

Tools: Debezium (open-source CDC for many databases), Fivetran, Airbyte, AWS DMS.

## 18.9 Data Warehouses

A data warehouse is a columnar database optimized for analytical queries. Modern warehouses also store vectors, JSON, and run ML inside.

**Snowflake.** Cloud-native warehouse with strong ecosystem. Cortex provides LLM and vector capabilities inside the warehouse.

**BigQuery.** GCP's serverless warehouse. ML and vector search integrated. Strong for analytics-heavy workloads.

**Databricks.** Lakehouse architecture combining data lake and warehouse. Strong for ML, integrated with MLflow.

**Redshift.** AWS warehouse. Mature, integrated with AWS ecosystem.

**ClickHouse, DuckDB, StarRocks.** Open-source columnar engines for analytics, often used alongside cloud warehouses.

For GenAI, the warehouse holds: interaction logs, eval results, cost metrics, training data, model usage analytics.

## 18.10 Delta Lake, Iceberg, Hudi

Open table formats turn object storage into ACID-compliant tables. They underpin lakehouse architectures.

**Delta Lake.** Databricks-led; broadly compatible.

**Apache Iceberg.** Vendor-neutral, fastest growing adoption.

**Apache Hudi.** Strong on streaming and upserts.

For GenAI: store raw documents, chunks, embeddings, and interaction logs as Iceberg/Delta tables on object storage. Query from any engine (Spark, Trino, Snowflake, DuckDB). Cheap, durable, query-engine-independent.

## 18.11 The Modern Data Stack for GenAI

```
   Sources (databases, APIs, files, user events)
        |
   Ingest (Fivetran, Airbyte, Kafka, custom)
        |
   Lake (S3 + Iceberg / Delta) <-- raw layer
        |
   Transform (dbt, Spark, SQL warehouses)
        |
   Warehouse (Snowflake, BigQuery, Databricks)
        |
   Specialized stores
        +-- Vector DB (Pinecone, Qdrant, pgvector)
        +-- Feature store / context store
        +-- Search engine (Elasticsearch, OpenSearch)
        |
   Applications (chat, agents, search)
```

## 18.12 dbt — Transformation as Code

dbt (data build tool) compiles modular SQL into transformation pipelines. Treat data transformations like code: version, test, document, deploy.

For GenAI, dbt is the standard for transforming raw event data into analytics tables: per-day token consumption by model, eval scores by prompt version, user funnels through AI features.

## 18.13 Data Quality and Testing

Data quality issues silently destroy AI products. The same care applied to code tests must be applied to data.

Tools and patterns:
- **dbt tests** for inline assertions (unique, not null, accepted values, custom).
- **Great Expectations, Soda** for expectation-driven data quality.
- **Schema registries** for event payloads.
- **Data contracts** between producers and consumers.

For GenAI: assert that retrieved chunks have valid embeddings, that no PII has slipped into a public corpus, that eval scores are within historical bands.

## 18.14 Data Governance

Governance covers: who owns each dataset, who can access what, how data is classified, how long it is retained, how access is audited.

For GenAI specifically:
- **Document provenance.** Track where every document in the RAG corpus came from.
- **Access control on retrieval.** Users only see what they are authorized to see.
- **Retention.** User conversations and uploads may be subject to deletion requests (GDPR).
- **Audit logs.** Every access to sensitive data recorded.
- **Lineage.** Trace any model output back through the data that produced it.

Tools: Apache Atlas, DataHub, Unity Catalog (Databricks), Collibra, Alation.

## 18.15 Data Pipelines for RAG

```
   Sources (Confluence, SharePoint, S3, DBs)
        |
   Connectors / sync
        |
   Raw lake (with metadata: source, timestamp, ACL)
        |
   Parsers (PDF, HTML, Office)
        |
   Chunker (structure-aware)
        |
   Enricher (summaries, tags, language detection)
        |
   Embedder (batched, GPU)
        |
   Vector DB + metadata store
        |
   BM25 index (optional)
        |
   Knowledge graph (optional)
        |
   Refresh scheduler (incremental or full)
```

Every step needs monitoring: throughput, error rate, queue depth, end-to-end freshness.

## 18.16 Real-Time vs Batch Embedding

**Batch embedding.** Cheaper per unit. Use for large corpora and nightly refresh.

**Real-time embedding.** Sub-second updates when a document changes. Use for high-freshness needs (live chat, monitoring dashboards).

Most production systems use both: bulk batch for initial load, streaming for ongoing changes.

## 18.17 Tool Comparison

| Tool | Category | Strength | Weakness |
|---|---|---|---|
| Kafka | Streaming | Battle-tested, durable | Operationally heavy |
| Redpanda | Streaming | Lighter ops | Less ecosystem |
| Snowflake | Warehouse | Ecosystem, ergonomics | Cost at scale |
| BigQuery | Warehouse | Serverless, ML | GCP-only |
| Databricks | Lakehouse | ML-native | Cost complexity |
| Iceberg | Table format | Vendor-neutral | Less mature than Delta |
| dbt | Transformation | SQL-as-code | Not for streaming |
| Airbyte | Ingest | Open source, many connectors | Some connectors immature |
| Fivetran | Ingest | Managed, reliable | Cost |

## 18.18 Tradeoffs

| Choice | Win | Cost |
|---|---|---|
| Streaming-first | Freshness, real-time features | Complexity |
| Batch-first | Simplicity, cost | Latency |
| Cloud warehouse | Ergonomics, ecosystem | Cost |
| Lakehouse | Open formats, flexibility | More tooling |
| Managed ingest (Fivetran) | Hands-off | Subscription cost |
| DIY connectors | Cheap, custom | Maintenance burden |

## 18.19 Scaling Challenges

- Embedding throughput at billion-document scale.
- Schema evolution in long-running event streams.
- Cross-region replication for global products.
- Cost as data volume grows.
- Compliance with retention and residency.

## 18.20 Security Concerns

- Encryption at rest and in transit.
- Tenant isolation for multi-tenant SaaS.
- Access control on warehouses and vector DBs.
- PII detection and redaction in pipelines.
- Audit logging for sensitive datasets.

## 18.21 Cost Optimization

- Tier hot/warm/cold storage.
- Use spot for batch jobs.
- Compress aggressively.
- Right-size warehouse compute.
- Cache query results.
- Quantize vectors.

## 18.22 Interview Questions

- Compare ETL and ELT in the context of GenAI.
- Walk through a CDC pipeline that keeps a vector index in sync.
- When do you choose Kafka vs RabbitMQ?
- Design a data governance model for an enterprise RAG.
- How do you handle GDPR delete requests across a vector DB?

## 18.23 Hands-on Exercises

1. Sketch the full data pipeline for a customer support knowledge base with 100k documents updated daily.
2. Design the schema for an interactions table that captures every chat with cost, latency, and quality signals.
3. Plan deletion handling: a user requests deletion; what must change across raw, warehouse, vector, and backups?

## 18.24 Common Mistakes

- Treating documents as immutable; failing to handle updates.
- No schema registry; producers and consumers drift silently.
- Skipping data quality tests; bad data poisons retrieval.
- Conflating ingestion and embedding; one schedule does not fit both.
- No retention policy; storage grows forever.

## 18.25 Enterprise Best Practices

Lakehouse with Iceberg or Delta from the start. dbt for transformations. Schema registry for streams. Data contracts between producer and consumer teams. Governance catalog. Per-dataset owner and SLA. Quarterly cost and quality reviews.
