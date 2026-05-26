# Chapter 14 — Performance Engineering

## 14.1 Concept Explanation

Performance engineering is the discipline of making systems faster and more efficient under load. It is adjacent to capacity planning but different: capacity planning answers "how much do we need?"; performance engineering answers "how do we need less?"

Performance work has direct economic value. A 2x performance improvement halves required capacity, halving costs. It also improves user experience and resilience — faster systems have more headroom for surges.

A senior SRE does not guess where to optimize. They measure, find the dominant bottleneck, fix it, and measure again. Premature optimization is famously the root of evil; targeted optimization with measurement is one of the highest-leverage activities in production.

## 14.2 Latency vs Throughput

These two performance dimensions trade off:

**Latency** — how long one operation takes. Measured in milliseconds. Critical for interactive services (chat, search, checkout).

**Throughput** — how many operations per second. Measured in QPS. Critical for batch and high-traffic systems.

Optimizations that improve one often hurt the other:
- Batching improves throughput, hurts per-request latency.
- Parallelization improves latency, may reduce throughput per core.
- Caching helps both, but consistency suffers.
- Larger machines may improve both, at cost.

Be explicit about which you are optimizing.

## 14.3 Percentiles vs Averages

Average latency hides the experience of users at the tail. If average is 100ms but p99 is 5 seconds, 1% of users have a terrible experience.

Always think in percentiles:
- **p50 (median)** — typical experience.
- **p90** — common but slower.
- **p95** — uncommon.
- **p99** — tail.
- **p999** — rare tail.
- **max** — the worst case.

User experience is often determined by p95-p99. A page that loads 95 components must wait for the slowest — effectively the p99 of each.

## 14.4 The Performance Hierarchy

Performance issues come from this hierarchy of causes, roughly in order of frequency:

1. **Algorithmic issues** — O(n²) where O(n) would do.
2. **I/O issues** — too many database calls, no batching, no caching.
3. **Lock contention** — synchronization bottlenecks.
4. **GC pressure** — too many allocations.
5. **Network round trips** — chatty protocols.
6. **CPU-bound code** — actual compute being slow.
7. **Memory pressure** — paging, cache misses.
8. **Resource exhaustion** — file descriptors, connections, threads.

Start at the top. Most performance problems are algorithmic or I/O. Real CPU bottlenecks are rarer than people think.

## 14.5 Profiling Tools

You cannot optimize what you cannot see. Profilers reveal where time goes.

**CPU profilers** — sample stack traces over time. Identify hot functions.
- **perf** (Linux) — sampling profiler, low overhead.
- **eBPF profilers** (BCC, bpftrace, Parca, Pyroscope) — modern, production-safe.
- **Flame graphs** (Brendan Gregg) — visualization of where CPU time went.
- **Async profiler** (Java) — JVM profiling without bias.
- **py-spy** (Python), **pprof** (Go), **clinic** (Node.js).

**Memory profilers** — heap snapshots, allocation tracking.
- **heaptrack** (C++), **memray** (Python), **pprof** (Go).

**I/O / network profilers** — traces of disk and network operations.
- **iostat**, **iotop**, **tcpdump**, **Wireshark**, **bpftrace**.

**Database profilers** — query plans, slow query logs.

The pattern: profile in production-like conditions; aggregate over time; visualize as flame graphs; iterate.

## 14.6 Continuous Profiling

Profilers historically ran in development. **Continuous profiling** runs them in production constantly, at low sample rates. Tools:
- **Pyroscope** (now part of Grafana).
- **Parca**.
- **Polar Signals Cloud**.
- **Datadog Continuous Profiler**.
- **Google Cloud Profiler**.

Continuous profiling lets you answer "what was the CPU doing during last night's slow period?" — even after the moment is gone.

## 14.7 The Performance Engineering Workflow

1. **Define the goal.** What metric, what target. "Reduce p99 latency from 300ms to 100ms."
2. **Measure baseline.** Current value with confidence interval.
3. **Identify the bottleneck.** Profile, look at traces, examine logs.
4. **Form hypothesis.** "Database query X is the bottleneck because it does N+1 fetches."
5. **Make change.** Smallest fix that addresses the hypothesis.
6. **Measure impact.** Did the metric improve?
7. **Iterate** or move to next bottleneck.

Without this discipline, performance work degenerates into "I think this might be faster."

## 14.8 The Top Bottlenecks in Practice

A non-exhaustive list of common bottlenecks:

**Database N+1.** Loading a list, then a query per item. Fix: join or batch fetch.

**Missing index.** Full table scan when index would suffice. Fix: add index, watch for write impact.

**Synchronous calls to slow services.** Use async/parallel where possible.

**Chatty protocols.** Many small round-trips. Fix: batch, use gRPC, use HTTP/2.

**JSON serialization.** Often a hidden cost. Fix: smaller payloads, binary protocols.

**Logging.** Excessive logging blocks request paths. Fix: async logging, reduce volume.

**Lock contention.** Threads waiting on locks. Fix: smaller critical sections, lock-free data structures.

**Cold caches.** Cache miss on every request. Fix: warm caches at startup.

**TLS handshake overhead.** Per-connection. Fix: connection pooling, HTTP/2.

**Inefficient regex / parsing.** Catastrophic backtracking, slow tokenizers. Fix: rewrite.

## 14.9 Database Performance

Databases are the most common bottleneck. Key levers:

- **Indexes** — make reads fast, slow writes. Tune based on query patterns.
- **Query plans** — examine via `EXPLAIN`. Look for full scans.
- **Connection pooling** — reuse connections. PgBouncer, RDS Proxy.
- **Read replicas** — offload reads.
- **Caching** — Redis in front of DB.
- **Partitioning** — large tables broken into manageable pieces.
- **Vacuum / maintenance** — PostgreSQL needs regular vacuuming.
- **Avoid SELECT *** — fetch what you need.
- **Batch writes** — multi-row inserts.

## 14.10 Cache Performance

Caches multiply effective performance. Patterns:

- **Cache-aside** — application checks cache, falls back to source, populates cache. Standard.
- **Write-through** — writes go to cache and source synchronously. Strong consistency, slower writes.
- **Write-behind** — writes go to cache, async to source. Fast, risk of loss.
- **Read-through** — cache transparently fetches misses. Simpler app code.

Pitfalls:
- **Stampede** — when a cache entry expires, all clients hit the source simultaneously. Fix with locks or background refresh.
- **Invalidation** — keeping cache consistent with source is hard. TTL is the simplest answer.
- **Hot keys** — one key gets all traffic. Fix with sharding or replication.

## 14.11 Network Performance

Network is often slower than people remember. Round-trip times matter:

- **Same machine** — microseconds.
- **Same data center** — sub-millisecond.
- **Same region, different AZ** — 1-5ms.
- **Cross-region** — 30-200ms+.
- **Cross-continent** — 50-300ms+.

Designs that ignore network latency look fast in dev (everything local) and slow in prod (everything remote).

Optimizations:
- **HTTP/2** for multiplexing and header compression.
- **HTTP/3** for connection migration and reduced handshake.
- **gRPC** for compact binary RPC.
- **Connection pooling** to amortize handshake.
- **Compression** (gzip, brotli, zstd).
- **CDN** for static assets and cached responses.

## 14.12 Memory and GC

Garbage collection can dominate latency in JVM and Node.js services.

Symptoms:
- Periodic latency spikes correlated with GC pauses.
- Memory grows over time (leak or just heap growth).
- High CPU on GC threads.

Fixes:
- Tune GC algorithm (G1, ZGC, Shenandoah for Java; V8 flags for Node).
- Reduce allocation pressure (object pooling, primitive types).
- Increase heap (longer between GCs).
- Off-heap storage for large caches.

For Go and Rust, GC is less of a concern but memory layout still matters.

## 14.13 The Latency Numbers Every Engineer Should Know

Approximate orders of magnitude:
- L1 cache reference — 1 ns.
- L2 cache reference — 4 ns.
- Main memory reference — 100 ns.
- SSD random read — 100 µs.
- HDD seek — 10 ms.
- Same datacenter RTT — 0.5 ms.
- Cross-continent RTT — 150 ms.

When designing, mentally place each operation on this scale. A "small DB query" includes a network round-trip.

## 14.14 Real-World Use Cases

- A team's API was 500ms p99. Profiling showed 200ms in JSON serialization of a deeply nested response. Trimmed unnecessary fields; p99 dropped to 250ms.
- A database query was slow because the index was on (created_at) but the query filtered by (user_id, created_at). Added composite index; query went from 5s to 5ms.
- A service was OOM-killed nightly. Memory profile showed an in-memory cache without size limit. Bounded it; problem solved.

## 14.15 Production Architecture for Performance Work

```
   Continuous profiling (Pyroscope, Parca) running in prod
        |
   Per-service performance dashboards (latency percentiles, throughput)
        |
   Distributed tracing for per-request breakdown
        |
   Slow query logs for databases
        |
   Performance regression detection (CI benchmarks)
        |
   Quarterly performance reviews per critical service
```

## 14.16 Tradeoffs

| Choice | Win | Cost |
|---|---|---|
| Caching | Speed | Consistency complexity |
| Batching | Throughput | Latency |
| Connection pooling | Less handshake | More memory |
| Bigger machines | Faster | Cost |
| Microservices | Independent scaling | Network overhead |
| Monolith | Less network | Coarse scaling |

## 14.17 Scaling Challenges

- Performance varies with load — a fast system at 100 QPS may be slow at 10,000.
- Multi-region adds cross-region latency.
- Hot paths shift with feature releases.
- Tail latency requires statistical thinking, not averages.

## 14.18 Security

- Performance fixes can introduce security holes (skipping validation).
- Caching sensitive data needs encryption and TTL discipline.
- DoS resistance overlaps with performance (slow attacks).

## 14.19 Deployment Guide

To stand up performance engineering:
1. Deploy continuous profiling.
2. Add per-service performance dashboards.
3. Enable slow query logs on databases.
4. Set up CI performance benchmarks for critical paths.
5. Quarterly performance review per Tier-0 service.

## 14.20 Monitoring Strategy

- Latency percentiles per endpoint.
- Throughput per service.
- CPU and memory utilization.
- GC time (where applicable).
- Cache hit rate.
- Database query latency.

## 14.21 Cost Optimization

Performance improvements directly reduce cost — fewer servers needed for the same work. Track $/request as a metric; aim to reduce it over time.

## 14.22 Interview Questions

- *Walk me through how you'd debug a latency regression.*
- *Explain the difference between latency and throughput.*
- *What's a flame graph?*
- *Compare cache-aside and write-through.*
- *Why do percentiles matter more than averages?*

## 14.23 Hands-on Exercises

1. Profile a small service using a flame graph. Identify the top three hot functions.
2. Take a slow database query. Use `EXPLAIN` to identify the issue. Propose an index or rewrite.
3. Estimate the dollar cost of a 30% latency improvement on a service at scale.

## 14.24 Common Mistakes

- Optimizing without measuring (premature optimization).
- Optimizing the wrong layer (CPU when I/O is the bottleneck).
- Average latency thinking (ignoring tails).
- Cache as panacea (caches add bugs).
- Skipping profiling because "it's complex" (profilers are easier than ever).

## 14.25 Enterprise Best Practices

Continuous profiling in production. Performance budgets per critical service. CI performance regression gates. Quarterly performance reviews. Training on flame graphs and profiling. Cost-per-request tracked alongside SLOs.
