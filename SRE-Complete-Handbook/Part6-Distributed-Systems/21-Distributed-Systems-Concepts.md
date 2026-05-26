# Chapter 21 — Distributed Systems Concepts

## 21.1 Concept Explanation

A distributed system is one in which components running on networked computers coordinate to achieve a goal. Every production system worth running is distributed. The problems unique to distributed systems — partial failure, asynchrony, network unreliability, ordering — are what most SRE work eventually addresses.

This chapter covers the foundational concepts every SRE must internalize. The mathematics is light; the intuition is heavy. Senior SREs do not need to prove Paxos correct; they do need to predict how systems behave under failure.

## 21.2 The Fallacies of Distributed Computing

Sun Microsystems engineer Peter Deutsch listed assumptions everyone makes and that everyone is wrong about:

1. The network is reliable.
2. Latency is zero.
3. Bandwidth is infinite.
4. The network is secure.
5. Topology doesn't change.
6. There is one administrator.
7. Transport cost is zero.
8. The network is homogeneous.

Every one of these is false. Designing as if they are true causes outages. The fallacies are the implicit theme of distributed systems education.

## 21.3 The Two Generals Problem

Two armies must coordinate an attack. They can only communicate via messengers, who can be captured. Can they reach guaranteed agreement?

Answer: no. Any protocol can lose its final message. The receiver does not know if the sender knows the receiver received.

Implication: in an asynchronous network, perfect agreement is impossible. Real systems use *good enough* protocols (TCP with acks and retries), accept some probability of error, and design around it.

This is why two-phase commit is fragile, why idempotency matters, and why distributed transactions are hard.

## 21.4 CAP Theorem

In the presence of a network partition, you must choose between:
- **Consistency (C)** — every read sees the latest write.
- **Availability (A)** — every request gets a response.

You cannot have both during a partition. PACELC adds: even when there is no partition, you trade Latency for Consistency.

In practice:
- Most distributed databases are AP (return possibly stale data during partitions) or CP (refuse writes during partitions).
- "CA without P" is a single-machine database.
- Pure CA is impossible because partitions happen.

CAP is often misapplied. The honest summary: when networks break, your system has to choose what to do — refuse, or serve stale.

## 21.5 Consistency Models

A spectrum from strong to weak:

**Linearizability (strong consistency).** Operations appear to happen in some total order consistent with real time. Behaves like a single machine. Expensive.

**Sequential consistency.** Operations appear in some total order, not necessarily real-time order. Common in concurrent programming.

**Causal consistency.** If A causes B, all observers see A before B. Independent operations may be seen in different orders.

**Eventual consistency.** Given no new writes, all replicas converge. Most permissive, cheapest. Default for many distributed databases (DynamoDB, Cassandra).

**Read-your-writes.** A client always sees its own writes (but maybe not others').

**Monotonic reads.** A client never sees time go backward.

Each model is a tradeoff. Choose deliberately; do not stumble into eventual consistency and be surprised by anomalies.

## 21.6 The CAP Choices in Real Systems

| System | Choice |
|---|---|
| Spanner | CP (with TrueTime) |
| Cassandra | AP (tunable) |
| DynamoDB | AP by default, CP optional |
| MongoDB | CP by default, AP optional |
| ZooKeeper / etcd | CP |
| Redis (replicated) | AP |
| PostgreSQL (multi-replica) | CP |

Knowing where your data store sits on CAP saves you from "but we thought this was consistent" incidents.

## 21.7 Distributed Consensus

Some operations need agreement across nodes: leader election, configuration changes, exclusive resource allocation. Consensus protocols solve this.

**Paxos** — the classical algorithm. Notoriously hard to understand.

**Raft** — designed to be understandable. Now dominant.

**Multi-Paxos, Fast Paxos, EPaxos** — variants.

You will not implement these. You will use systems that do: etcd, ZooKeeper, Consul. Understanding the cost helps:
- Consensus has a majority requirement; need 2f+1 nodes to tolerate f failures.
- Latency = at least one round trip across the majority.
- Throughput limited by network and leader bottleneck.

## 21.8 Leader Election

In a cluster, one node typically does write coordination (the leader). When the leader fails, a new one must be elected.

Patterns:
- **Raft-based** — etcd, Consul, modern systems.
- **Bully algorithm** — older, less common.
- **External coordination** — leader picked via etcd/ZooKeeper.

Failures during leader election: split-brain (two leaders), no leader (writes stall). Real systems guard against both.

## 21.9 Quorums

A quorum is a set of nodes needed to perform an operation. Most often: majority.

For N replicas:
- Write to W, read from R.
- If W + R > N, you get strong consistency.
- If W + R ≤ N, you can read stale data.

Cassandra, DynamoDB, and other AP systems expose tunable W and R. You choose the tradeoff per query.

## 21.10 Replication

**Synchronous replication.** Writes wait for replicas. Strong consistency. Latency penalty.

**Asynchronous replication.** Writes return immediately; replicas catch up later. Lower latency. Data loss risk.

**Semi-synchronous.** Wait for at least N replicas (N < total). Compromise.

**Multi-master replication.** Writes can go to any node. Conflict resolution needed.

Replication is how you survive node failures. The cost is latency, complexity, or both.

## 21.11 Sharding

Splitting data across nodes to scale beyond one machine.

Approaches:
- **Range sharding** — shard by key range. Hotspots if traffic is uneven.
- **Hash sharding** — shard by hash. Even distribution; range queries hard.
- **Consistent hashing** — minimizes redistribution when nodes change.
- **Directory-based** — explicit shard map. Flexible, requires lookup.

Sharding decisions are hard to undo. Plan keys early.

## 21.12 The CAP-Latency Triangle

A useful refinement: in normal operation, you trade Consistency for Latency (PACELC). During partitions, you trade Consistency for Availability (CAP).

Most production systems sit at PA/EL — eventually consistent for partitions and latency optimized in normal operation. Some sit at PC/EC — strong consistency at cost of higher latency.

## 21.13 Time in Distributed Systems

Clocks drift. NTP keeps them roughly in sync but not perfectly. Important consequences:
- Cannot rely on wall clock for ordering events.
- Logical clocks (Lamport timestamps, vector clocks) provide ordering without physical time.
- Google Spanner uses TrueTime with hardware-bounded uncertainty.

A common production bug: code assumes time is monotonic across machines. It is not.

## 21.14 Idempotency Revisited

In a distributed system, you cannot distinguish "operation succeeded but the response was lost" from "operation failed." The only safe behavior under uncertainty is retry. Retry only works if operations are idempotent.

Make operations idempotent:
- Use natural idempotency where possible (SET, not INCREMENT).
- Add idempotency keys (request IDs) so duplicate requests are detected.
- Design retry-safe APIs.

Without this, retries cause duplicates: double charges, duplicate emails, double sends.

## 21.15 Backpressure and Flow Control

Producers can outrun consumers. Without flow control, the system buffers until OOM.

Mechanisms:
- TCP-level flow control.
- HTTP 429 Too Many Requests.
- Bounded queues that reject when full.
- Streaming protocols with credits (HTTP/2, WebSocket).

Designing for backpressure is the difference between graceful degradation and catastrophic failure.

## 21.16 Real-World Use Cases

- A "distributed transaction" using two-phase commit hung the database for hours when one participant failed. Replaced with sagas (compensating actions) and the problem went away.
- A team used wall-clock timestamps to order events. Daylight saving change caused 1 hour of duplicates. Switched to logical clocks.
- A multi-master setup created conflicting writes during a brief partition. Conflict resolution policy (last-write-wins) silently lost data. Moved to CRDTs.

## 21.17 Tradeoffs

| Choice | Win | Cost |
|---|---|---|
| Strong consistency | Predictable | Latency, availability hit |
| Eventual consistency | Available, fast | Anomalies |
| Sync replication | No data loss | Latency |
| Async replication | Fast writes | Data loss risk |
| Range shards | Range queries | Hotspots |
| Hash shards | Even spread | No range queries |

## 21.18 Common Failure Modes

- **Split brain** — two leaders, conflicting writes.
- **Network partition** — half of the cluster cannot see the other half.
- **Cascading failure** — one component slows down, retries amplify load on others.
- **Thundering herd** — cache expires, all clients hit source at once.
- **Slow node** — one node responds slowly, drags down the quorum.

Chapter 22 covers these in depth.

## 21.19 Interview Questions

- *Explain CAP and PACELC.*
- *Compare strong and eventual consistency.*
- *What is a quorum?*
- *Walk through how Raft elects a leader.*
- *Why is two-phase commit fragile?*

## 21.20 Hands-on Exercises

1. For a system you know, identify where it sits on the consistency spectrum.
2. List three places where time assumptions could cause bugs in your system.
3. Design idempotency keys for a payment API.

## 21.21 Common Mistakes

- Assuming "the network is reliable" until it isn't.
- Using wall clock for ordering.
- Optimistic distributed transactions.
- Retries without idempotency.
- Mixing consistency models across one product.

## 21.22 Enterprise Best Practices

Train all engineers in distributed systems basics. Document consistency model per data store. Idempotency built into APIs. Time handled via NTP plus logical ordering where needed. Quorum choices documented per database. Conflict resolution explicit, not implicit.
