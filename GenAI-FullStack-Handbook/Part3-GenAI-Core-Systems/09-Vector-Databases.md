# Chapter 9 — Vector Databases

## 9.1 Concept Explanation

A vector database stores high-dimensional vectors (typically embeddings, see Chapter 8) and answers a single question fast: *given this query vector, what are the most similar vectors I have?* At small scale, you could iterate over every vector and compute similarity. At a billion vectors, that is impossible in interactive time. Vector databases use approximate nearest neighbor (ANN) algorithms to find near-best matches in milliseconds.

A vector DB is not a general database with vectors added on. It is a specialized index optimized for nearest-neighbor search, often paired with conventional fields (filters, metadata) so you can scope results ("documents from this user, in this language, embedded within the last 30 days, semantically similar to this question").

## 9.2 ANN — Why We Cannot Search Exactly

Exact nearest-neighbor search requires comparing the query against every vector in the dataset — O(n) per query. At a million vectors, this is a few seconds; at a billion, it is minutes. Unacceptable for online use.

ANN trades exactness for speed. The algorithm returns the *approximate* nearest neighbors, typically with 95–99% recall (the fraction of true top-k that appear in the returned top-k). For semantic search, this is plenty — the user does not care whether you returned the absolute best match or one of the top three.

## 9.3 HNSW — The Dominant Index

Hierarchical Navigable Small World (HNSW) is the most widely used ANN algorithm in production. The intuition: build a multi-layer graph where each node connects to a small set of nearby neighbors and a few far-away "shortcut" neighbors. To search, start at the top layer (sparse, fast traversal across the dataset), descend layer by layer, refining at each level until you reach the target neighborhood.

Strengths: very fast queries, high recall, supports updates. Weaknesses: high memory usage (the graph structure plus the vectors themselves all in RAM is expensive at billion-scale), slower build times than some alternatives.

HNSW is what powers Qdrant, Weaviate's default index, pgvector's `hnsw` index, Milvus's HNSW backend, and Pinecone's serverless offering's primary algorithm.

## 9.4 IVF — Inverted File Index

IVF partitions the vector space into clusters using a clustering algorithm (typically k-means). At query time, you find the closest few clusters and search only within them.

Strengths: lower memory, faster builds, good for batch workloads. Weaknesses: lower recall than HNSW at equivalent speed, recall drops on edge-case queries (when the relevant vector happens to be in a cluster you did not search).

IVF is often combined with **PQ (Product Quantization)** — compressing each vector by splitting it into sub-vectors and encoding each with a small codebook. IVF-PQ enables billion-scale indices that fit in modest memory.

## 9.5 PQ and Quantization

Product Quantization compresses vectors massively. A 768-dim float32 vector (3 KB) becomes a few dozen bytes. At a billion vectors, this is the difference between 3 TB of RAM and 30 GB.

The cost is quality loss. PQ is lossy. Recall drops, especially for nuanced queries. The mitigation is *re-ranking*: retrieve more candidates than you need using the compressed index, then re-score the top candidates with the full-precision vectors.

Scalar quantization (8-bit per dimension) and binary quantization (1-bit per dimension, with bit-level Hamming distance) are simpler alternatives gaining traction. Modern embedding models like cohere-binary and OpenAI text-embedding-3 with binary support enable order-of-magnitude storage reductions.

## 9.6 Memory Layout and Internals

A vector index must store:
- The vector data (full precision, quantized, or both).
- The index structure (graph nodes, cluster centroids, posting lists).
- Metadata (per-vector fields, ACL tags, timestamps).
- Indexes on metadata for filtering.

Memory-resident indices are fastest. Disk-resident indices (DiskANN, Vamana) trade query latency for capacity. SSD-resident with intelligent caching is the modern compromise.

Per-vector metadata is usually stored alongside or in a paired relational table. The vector DB joins or filters by metadata at query time. Filter-first vs vector-first ordering depends on filter selectivity — covered below.

## 9.7 Major Vector Databases

**Pinecone** — managed-only, serverless and pod-based offerings. Strong ergonomics, mature filtering, fast. Cost can climb at large scale.

**Weaviate** — open-source and managed cloud. Schema-aware (objects, not just vectors). Built-in hybrid search.

**Milvus** — open-source, built for billion-scale. Multiple index backends. Zilliz is the managed offering.

**Qdrant** — open-source and managed cloud. Rust-based, high performance, strong filtering, payload-rich.

**Chroma** — developer-friendly, embedded or hosted. Great for prototypes and small to medium scale.

**pgvector** — Postgres extension. Vectors as a column. Excellent when you already have Postgres and want vector search alongside relational data. HNSW and IVF indexes supported.

**Elasticsearch / OpenSearch** — keyword search engines that now support vectors. Strong hybrid search, mature ops, but vector performance trails specialized DBs at extreme scale.

**Redis Vector Search** — Redis with vector capabilities. Fast for small in-memory indices. Limited at large scale.

**Vespa** — Yahoo's heavyweight search engine, now open-source. Strong for combined ranking workloads. Steeper learning curve.

**LanceDB, Marqo, Vald, Turbopuffer** — newer entrants with various specializations.

## 9.8 Comparison Table

| DB | Hosting | Index | Hybrid | Filtering | Scale Sweet Spot |
|---|---|---|---|---|---|
| Pinecone | Managed | HNSW + proprietary | Yes (sparse-dense) | Strong | Production, hands-off |
| Weaviate | OSS + managed | HNSW | Yes | Strong | Medium to large |
| Milvus / Zilliz | OSS + managed | Many | Yes | Strong | Billion-scale |
| Qdrant | OSS + managed | HNSW | Yes | Very strong | Medium to large |
| Chroma | OSS + managed | HNSW | Limited | Basic | Prototype to medium |
| pgvector | Postgres | HNSW + IVF | Via FTS | Excellent (SQL) | Small to medium |
| OpenSearch | OSS + managed | HNSW | Yes | Excellent | Medium to large |
| Redis | OSS + managed | Flat + HNSW | Yes | Decent | Small in-memory |
| Vespa | OSS + managed | Many | Yes | Excellent | Large with ranking |

There is no universal best. Pick by what you already operate, what scale you target, and how rich your filtering needs are.

## 9.9 Hybrid Search

Pure vector search misses exact matches. If the user asks for "ERROR-4521" (an error code), embedding might not retrieve the exact document. Hybrid search combines vector (semantic) and BM25 (keyword) scores, ranked together.

Two approaches:
- **Score fusion** — run both searches, combine scores with weights or Reciprocal Rank Fusion (RRF).
- **Sparse-dense hybrid** — store both a sparse vector (BM25-like) and a dense vector per document; the index handles both.

Hybrid search routinely outperforms pure vector or pure keyword on real-world retrieval benchmarks. Most production RAG systems use it.

## 9.10 Filtering Strategies

Real queries are scoped: "documents I have access to, in English, modified this year, semantically similar to my question." How filters compose with vector search materially affects performance.

- **Pre-filter (filter then search).** Restrict candidate set first, search vectors among those. Best when filter is highly selective (returns 0.1% of corpus). Worst when filter is barely selective (returns 90%) — you have done a lot of metadata work for nothing.
- **Post-filter (search then filter).** Search vectors first, drop non-matching results. Best when filter is loose. Risk: if filter is restrictive, your top-k may return zero matches.
- **Integrated filtering.** Modern vector DBs (Qdrant, Weaviate, Milvus) build the filter into the index traversal, achieving both correctness and performance.

This is one of the hard problems in vector DB design. Different DBs handle it differently; benchmark on your filter distribution.

## 9.11 Production Architecture

```
   Application
        |
   Vector DB client
        |
        +----- ANN query -----> Vector index (HNSW/IVF/etc.)
        +----- BM25 query -----> Sparse index (optional)
        +----- Metadata filter -> Filter index
        |
   Result fusion / re-ranking
        |
   Cross-encoder reranker (optional)
        |
   Top-k results back to application
```

Sharding and replication are standard at scale. Shard by document ID or by tenant. Replicate for read throughput and availability.

## 9.12 Scaling Challenges

- **Index build time** at billion-scale can take days. Plan incremental updates and periodic reindex windows.
- **Memory blowup** — HNSW indices are memory-hungry. Switch to disk-resident (DiskANN) when RAM cost dominates.
- **Hot keys** — when one tenant dominates traffic, sharding by tenant produces hot shards. Hash-shard or rebalance.
- **Refresh latency** — newly ingested documents may not appear in search for seconds to minutes. Document the SLA.

## 9.13 Replication and Sharding

- **Sharding** distributes vectors across nodes. Each query fans out to all shards, gathers candidates, and merges. More shards = more parallelism but more network coordination.
- **Replication** copies shards to multiple nodes. More replicas = more read throughput, higher availability, higher cost.
- **Consistency** — most vector DBs offer eventual consistency for new inserts. If you need read-after-write, check carefully.

## 9.14 Security Concerns

- Multi-tenant isolation: separate collections per tenant for hard isolation, or shared collection with per-document tenant_id and mandatory filter at the application boundary.
- Encryption at rest and in transit — table stakes; verify your provider supports it.
- Backup and restore — operational discipline; vector DBs are databases.
- Audit logging — track who queried what, especially in regulated industries.

## 9.15 Deployment Guide

For managed: pick by feature fit, sign up, allocate the right tier. For self-hosted: containerize, deploy on Kubernetes with persistent storage, expose via internal service, autoscale read replicas. Monitor disk, memory, p99 query latency, and replication lag.

## 9.16 Monitoring Strategy

- p50, p95, p99 query latency.
- Indexing throughput.
- Recall on a golden set (run nightly).
- Memory and disk usage trends.
- Per-tenant or per-namespace query volume.
- Error rate by query type.

## 9.17 Cost Optimization

- Quantization (PQ, scalar, binary).
- Lower-dim embeddings if quality allows.
- Tier old data to disk-resident indices.
- Right-size the managed tier — many teams over-provision.
- For Pinecone-class managed services, watch the storage/query split in pricing and optimize for whichever dominates.

## 9.18 Interview Questions

- Walk through HNSW vs IVF.
- How do filters interact with ANN search?
- Compare three vector databases.
- Design a multi-tenant retrieval system.
- How do you handle re-embedding at scale?

## 9.19 Hands-on Exercises

1. Sketch the index structure of HNSW on paper.
2. For a 100M-vector corpus with 1024-dim embeddings, plan a storage and memory budget.
3. Design a hybrid search ranker combining BM25 and cosine.
4. Plan a zero-downtime migration from one vector DB to another.

## 9.20 Common Mistakes

- Picking a vector DB before measuring traffic and filter patterns.
- Ignoring filter selectivity in performance planning.
- Skipping re-ranking when pure vector recall is mediocre.
- Treating the vector DB as a stable long-term contract — formats and APIs evolve fast.

## 9.21 Enterprise Best Practices

Encapsulate vector DB access behind an internal service so you can swap providers. Maintain a golden retrieval evaluation set and run it nightly. Plan capacity in tenants and growth rate, not in raw vectors. Document the recall and latency SLA per index. Treat reindex as a quarterly drill, not an emergency.
