# Chapter 24 — Caching and Storage Patterns

## 24.1 Concept Explanation

Caching makes things fast and cheap. Storage makes things durable. Both are deep topics; SREs need working fluency in both. This chapter is a pragmatic tour focused on the patterns that show up in production systems.

A clarifying frame: cache is performance. Storage is correctness. Confusing the two — using cache as a database, treating storage as fast enough — is the source of much pain.

## 24.2 Caching Strategies

**Cache-aside (lazy).** Application checks cache first; on miss, reads from source and populates cache. Most common.

**Read-through.** Cache transparently reads from source on miss. Application talks only to cache.

**Write-through.** Writes go to cache and source synchronously. Cache always consistent with source.

**Write-behind (write-back).** Writes go to cache; cache writes to source asynchronously. Fastest writes, risk of loss.

**Refresh-ahead.** Cache proactively reloads entries before expiry. Good for stable data.

Cache-aside is the default. Switch only with a specific reason.

## 24.3 Cache Invalidation

"There are only two hard problems in computer science: cache invalidation and naming things." — Phil Karlton.

Strategies:
- **TTL (Time-To-Live).** Entry expires after N seconds. Simplest. Stale tolerated for up to TTL.
- **Explicit invalidation.** App tells cache when source changes. Strong consistency, complex coordination.
- **Versioned keys.** Key includes version; new version on change. No invalidation needed; old entries become unreferenced.
- **Event-driven.** Cache subscribes to change events.

For most use cases, TTL is the right answer. Pick a TTL that matches the data's update rate.

## 24.4 Cache Hit Rate

The key metric. Hit rate = hits / (hits + misses).

High hit rate (>80%) = cache is doing its job.
Low hit rate (<50%) = cache may not be worth the operational cost; investigate why.

Improving hit rate:
- Larger cache.
- Longer TTLs (if consistency allows).
- Better cache key design.
- Pre-warming on startup.
- Adaptive TTLs based on data popularity.

## 24.5 Cache Stampede

When a popular cache entry expires, all clients miss simultaneously and hit the source. Source gets overwhelmed.

Defenses:
- **Locking** — only one client refreshes; others wait.
- **Probabilistic early expiration** — clients refresh slightly before TTL with low probability, spreading load.
- **Background refresh** — never expire; background process keeps cache fresh.
- **Stale-while-revalidate** — serve stale, asynchronously refresh.

Stampede is one of the top "cache caused an outage" patterns.

## 24.6 Hot Keys

One cache key gets disproportionate traffic. The node holding it becomes the bottleneck.

Defenses:
- **Client-side caching** — replicate hot keys in clients.
- **Local + remote two-tier cache.**
- **Sharded replicas** — replicate hot keys to multiple nodes.
- **Detect and route hot keys differently.**

Hot keys are unpredictable; designs should be robust to them.

## 24.7 Cache Storage Tiers

Modern apps often have multiple cache layers:

1. **Process memory** — fastest, smallest, per-instance.
2. **Local cache (sidecar)** — same node, shared across processes.
3. **Distributed cache (Redis, Memcached)** — shared across cluster.
4. **CDN** — geographically distributed.

Use the appropriate tier for each access pattern. Per-instance counters: process memory. Shared state: distributed cache. Static assets: CDN.

## 24.8 Redis

The dominant distributed cache. Beyond key-value:
- Strings, hashes, lists, sets, sorted sets.
- Pub/Sub messaging.
- Streams (Kafka-like).
- Lua scripting for atomicity.
- TTL on any key.
- Persistence (RDB snapshots, AOF logs).

Production patterns:
- **Standalone** — single node, simplest.
- **Replication** — primary + replicas for read scaling.
- **Sentinel** — automatic failover.
- **Cluster** — sharded for capacity.
- **Managed (ElastiCache, Memorystore)** — recommended for most.

Redis is more than a cache — it is a Swiss Army knife. Be careful about over-relying on a single instance.

## 24.9 Memcached

Older, simpler distributed cache. Key-value only, no persistence. Multi-threaded (Redis is single-threaded per node).

Use Memcached when you need pure ephemeral caching at scale. Use Redis when you need data structures, persistence, or features.

## 24.10 CDN Caching

CDNs cache static and (increasingly) dynamic content close to users.

- **Static caching** — images, JS, CSS. Long TTLs. Works for most sites.
- **Dynamic caching** — HTML, API responses. Requires careful cache key and invalidation.
- **Edge functions** — code at the edge (Cloudflare Workers, Fastly Compute@Edge, AWS Lambda@Edge).

A modern app's first 100ms can be largely the CDN handshake; tuning here pays off.

## 24.11 Storage Patterns

For durable data, the choices:

**Relational (Postgres, MySQL, Aurora, Spanner).** Best when:
- Strong consistency required.
- Complex queries.
- Schema enforcement valued.
- ACID transactions.

**Key-value (DynamoDB, Cassandra, Bigtable).** Best when:
- High throughput, low latency.
- Predictable access pattern (by key).
- Horizontal scale.

**Document (MongoDB, DynamoDB, Firestore).** Best when:
- Flexible schema.
- Nested data.
- App-driven queries.

**Time-series (Prometheus, InfluxDB, TimescaleDB).** Best when:
- Append-mostly, time-indexed.
- Aggregation over windows.

**Column-store (BigQuery, Snowflake, Redshift, ClickHouse).** Best when:
- Analytics, OLAP.
- Aggregations over large datasets.

**Object store (S3, GCS, Azure Blob).** Best when:
- Large files.
- Cheap durable storage.
- Append-only, immutable.

**Search (Elasticsearch, OpenSearch, Algolia).** Best when:
- Full-text search.
- Faceted filtering.

Most production systems use multiple. Avoid using one tool for all jobs.

## 24.12 Replication for Reliability

Replicate data so the loss of one node does not lose data.

Patterns:
- **Primary-replica** — one writer, many readers.
- **Multi-primary** — writes anywhere; conflict resolution needed.
- **Quorum** — write to W nodes, read from R nodes (Cassandra-style).

Synchronous replication (wait for replicas) gives durability but adds latency.
Asynchronous gives speed but risks loss if primary dies before sync.

Choose deliberately.

## 24.13 Backups

Disk failures, accidental deletes, data corruption — backups save you.

Practices:
- **Automated** — scheduled, no human in the loop.
- **Off-site** — different region or provider.
- **Encrypted** — at rest and in transit.
- **Tested restores** — quarterly at minimum.
- **Retention policies** — daily for a month, weekly for a year, monthly forever.

An untested backup is not a backup. Restore tests are mandatory.

## 24.14 Schema Migrations

Changing the shape of data in production. Risky.

Patterns:
- **Backward-compatible changes** — add columns, deprecate fields, drop later.
- **Online migrations** — change without downtime, often multi-step.
- **Shadow tables** — write to both old and new; switch reads when caught up; drop old.

Avoid:
- Destructive changes without backups.
- Schema changes deployed simultaneously with code changes.
- Long-running locks on large tables.

## 24.15 Real-World Use Cases

- A site put session data in Redis without persistence. A Redis restart logged out every user. Lesson: cache vs storage.
- An e-commerce site cached product prices for 1 hour. A price update did not propagate; customers got the old price. Lesson: TTL must match update rate.
- A schema migration locked a large table for hours. Service down. Lesson: online migrations.

## 24.16 Production Architecture

```
   Application
        |
   Process-level cache (local in-memory)
        |
   Redis (distributed cache, hot data)
        |
   Primary database (source of truth)
        +-- Read replicas (read scaling)
        +-- Async replicas (cross-region DR)
        +-- Daily backups to S3 (cross-region)
        |
   CDN for static and select dynamic content
        |
   Object storage for large blobs
```

## 24.17 Tradeoffs

| Cache pattern | Win | Cost |
|---|---|---|
| Cache-aside | Simple | App handles misses |
| Write-through | Consistent | Slower writes |
| Write-behind | Fast writes | Risk of loss |
| TTL invalidation | Simple | Stale tolerated |
| Explicit invalidation | Fresh | Coordination |

| Storage choice | Win | Cost |
|---|---|---|
| Postgres | Strong, expressive | Vertical scale limits |
| DynamoDB | Scale, low latency | Limited queries |
| S3 | Cheap, durable | Slow per-object access |
| Redis | Fast, flexible | Memory cost |

## 24.18 Scaling Challenges

- Cache stampede on popular keys.
- Hot keys imbalance.
- Replication lag bleeding into application behavior.
- Schema migrations on huge tables.
- Cross-region replication for global products.

## 24.19 Security

- Encryption at rest and in transit.
- Per-tenant isolation.
- Access controls and audit.
- PII handling (encryption, deletion).
- Backup encryption and access.

## 24.20 Deployment Guide

Choosing for a new product:
1. Pick a primary database (Postgres for most; DynamoDB for scale-first).
2. Add Redis for caching and ephemeral state.
3. Add object storage for files.
4. Plan replication and backup strategy.
5. Document data retention.
6. Plan first migration (you will need one).

## 24.21 Monitoring Strategy

For caches:
- Hit rate.
- Latency.
- Memory usage.
- Evictions.
- Hot key detection.

For storage:
- Replication lag.
- Backup success and restore test results.
- Storage growth rate.
- Query latency by type.
- Lock waits.

## 24.22 Cost Optimization

- Right-size caches (over-provisioning is common).
- Tier storage by access pattern (hot → SSD, cold → S3).
- Compress where possible.
- Set retention policies.
- Audit unused indexes.

## 24.23 Interview Questions

- *Compare cache-aside and write-through.*
- *How do you prevent cache stampede?*
- *When do you choose Postgres vs DynamoDB?*
- *Walk through a zero-downtime schema migration.*
- *How do you test backups?*

## 24.24 Hands-on Exercises

1. For a service, list every piece of state. Classify: cache, storage, durable.
2. Calculate the data loss exposure of each store (RPO).
3. Plan an online migration for an existing table.

## 24.25 Common Mistakes

- Cache as database (no persistence; expects durability).
- TTLs much longer than data lifetime.
- No cache invalidation strategy.
- One Redis for everything; failure takes everything down.
- No backup tests.
- Schema migrations not rehearsed.

## 24.26 Enterprise Best Practices

Document the data tier per service. Cache strategies reviewed during production readiness. Backups tested quarterly. Online migration playbooks. Per-service hit rate targets. Capacity planning for storage growth. Encryption mandatory.
