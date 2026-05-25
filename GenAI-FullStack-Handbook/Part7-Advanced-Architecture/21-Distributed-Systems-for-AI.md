# Chapter 21 — Distributed Systems for AI

## 21.1 Concept Explanation

A distributed system is one in which components running on networked computers coordinate to achieve a goal. AI systems are distributed by default: the model runs on one set of nodes, the vector DB on another, the application on a third, the message bus on a fourth. All the classical distributed-systems problems — consistency, ordering, failure handling, latency — apply.

This chapter applies distributed systems fundamentals to AI-specific architectures. Many design choices in earlier chapters trace back to these principles.

## 21.2 CAP Theorem

The CAP theorem states that a distributed system can guarantee at most two of:
- **Consistency.** Every read sees the most recent write.
- **Availability.** Every request gets a response (possibly stale).
- **Partition tolerance.** The system continues to operate despite network partitions.

In practice, partitions happen, so the choice is between consistency and availability during a partition.

For AI:
- Vector DBs typically choose availability — eventually consistent inserts are fine; the index lag is measured in seconds.
- Session state typically chooses consistency — losing chat history is worse than a brief unavailability.
- Caches typically choose availability — stale data is acceptable for a few seconds.

Knowing which side each component is on prevents surprises during failures.

## 21.3 PACELC

Refinement of CAP: even when there is no partition, systems trade Latency for Consistency. PACELC: if Partition, choose Availability or Consistency; Else, choose Latency or Consistency.

Production systems care about the "else" branch as much as the "P" branch. Strong-consistency reads are slower; eventual-consistency reads are faster. Pick deliberately.

## 21.4 Distributed Queues

Queues decouple producers and consumers, smooth bursts, and enable retries.

For AI:
- **Embedding jobs** in a queue, workers pull and process.
- **Agent steps** in a queue if long-running.
- **Eval runs** in a queue.
- **Inference requests** in a queue if traffic is bursty and latency tolerates queuing.

Properties to consider: ordering (FIFO?), at-least-once vs exactly-once delivery, durability, throughput.

## 21.5 Kafka — Streaming and Event Log

Kafka (Chapter 18) is more than a queue. It is a durable, ordered, partitioned log.

For AI specifically:
- **Event sourcing** of conversations — the conversation history is a log; the current state is derived.
- **Replayable streams** for backtesting new prompts against old data.
- **Audit trail** for compliance.

## 21.6 RabbitMQ

RabbitMQ is a mature broker with rich routing semantics — direct, topic, fanout, headers exchanges. Strong for task queues where Kafka's log-and-replay model is overkill.

For AI: task queues for embedding workers, agent step dispatch, batch eval jobs.

## 21.7 NATS

NATS is a lightweight, high-performance pub/sub messaging system. JetStream adds persistence. Lower operational footprint than Kafka.

For AI: low-latency inter-service eventing, especially in microservice architectures with many small messages.

## 21.8 Redis Streams

Redis with stream semantics. Simple, fast, suitable for small to medium scale. Limits show up at large persistent log scale; Kafka or Redpanda takes over.

## 21.9 Event Sourcing

Store every state-changing event as an immutable record. Current state is derived by replaying events.

For AI:
- Conversation history as event log.
- Agent step traces as event log.
- Eval result history as event log.

Benefits: full audit trail, replay for debugging, time-travel for state at any point.

Costs: storage, query complexity (snapshots help), schema evolution discipline.

## 21.10 CQRS — Command Query Responsibility Segregation

Separate the write path (commands that change state) from the read path (queries that read state). The two can have different models and scale independently.

For AI:
- Writes: ingestion pipeline updates documents.
- Reads: retrieval queries the vector DB.
- The vector DB is essentially a materialized read view of the document corpus.

Most production RAG systems implicitly use CQRS. Naming it clarifies design.

## 21.11 Eventual Consistency

In a distributed system without synchronous coordination, reads may temporarily see stale data. Eventual consistency means the system converges given enough time without new writes.

For AI:
- New documents may take seconds to appear in the vector index.
- New prompts may take minutes to propagate across all replicas.
- New cache entries may not be visible to all readers immediately.

Design with the staleness in mind. UIs can show "still indexing" states. Caches should have TTLs. Index refresh schedules should be documented.

## 21.12 Distributed Consensus

When multiple nodes must agree on something (leader election, configuration changes, transaction order), consensus protocols are used. Raft and Paxos are the foundational algorithms.

For AI infrastructure:
- Kubernetes uses etcd (Raft).
- Some vector DBs use Raft for replication.
- Service discovery often uses Raft-based stores.

You usually consume consensus as a feature, not implement it.

## 21.13 Distributed Tracing

Covered in Chapter 19. The fundamental tool for understanding distributed systems. Every request gets a trace ID; every span links into the trace. Without this, debugging multi-service flows is guesswork.

## 21.14 Idempotency

An idempotent operation produces the same result whether executed once or many times. Critical for retries, which are mandatory in any distributed system.

For AI:
- Embedding a document with a deterministic ID — idempotent.
- Posting a chat message with a client-provided ID — idempotent.
- Calling a "send email" tool — NOT idempotent without an idempotency key.

Design idempotent operations wherever possible. Where impossible, use explicit idempotency keys.

## 21.15 Backpressure

When a consumer cannot keep up with a producer, backpressure signals the producer to slow down. Without it, queues grow until something breaks.

For AI:
- Inference servers signal "queue full, reject" instead of accepting infinite requests.
- Embedding workers signal "behind" so ingestion slows.
- Streaming responses pause when client cannot consume tokens.

## 21.16 Circuit Breakers and Retries

A circuit breaker stops calling a downstream service after a threshold of failures. Better to fail fast than pile up timeouts.

Retries with exponential backoff and jitter handle transient failures without amplifying load.

For AI: LLM provider failures and rate limits are real. Wrap every external call with retries (idempotent-aware) and circuit breakers.

## 21.17 Bulkheads

Isolate resources so failure in one part of the system does not cascade. Separate thread pools per dependency, separate worker pools per priority class.

For AI: separate inference pools for free vs paid users so a free-tier surge does not impact paid customers.

## 21.18 Workflow Engines

For long-running, multi-step processes (agents, document ingestion, batch evals), workflow engines provide durability, retries, and visibility.

- **Temporal.** Durable workflows in mainstream languages. Strong for agentic systems.
- **Restate.** Newer alternative with similar guarantees and lower operational footprint.
- **Inngest.** Event-driven workflow platform, serverless.
- **AWS Step Functions, GCP Workflows, Azure Logic Apps.** Cloud-managed workflows.

Workflow engines often replace ad-hoc orchestration in agentic systems, turning fragile loops into durable resumable processes.

## 21.19 Microservices for AI

The decomposition for a typical AI system:
- Auth service.
- Chat service (handles SSE, conversation state).
- Retrieval service.
- LLM gateway.
- Tool services (one per major tool category).
- Embedding service.
- Eval service.
- Observability service.

Each scales independently, deploys independently, can be written in the most appropriate language.

The cost is operational: more services means more deployments, more on-call rotations, more inter-service contracts. Match decomposition to team size.

## 21.20 Scaling Inference Systems

Patterns for scaling inference horizontally:
- **Replica pools** per model, autoscaling on queue depth.
- **Tiered pools** — large model for hard queries, small model for easy ones.
- **Sharded by tenant or workload class.**
- **Regional clusters** to keep latency low.
- **Hot standby** for failover.

The hidden cost: cold starts on big models are slow. Always-on warm capacity is often necessary even at low traffic.

## 21.21 Distributed Memory Systems

Agent memory at scale becomes a distributed memory system. Properties needed:
- Low-latency reads (memory lookup is on the request critical path).
- Eventual consistency tolerated.
- Per-user partitioning.
- Multi-region replication for global users.

Implementations: Redis or DynamoDB for hot memory, vector DB for semantic memory, Postgres for structured profile data.

## 21.22 Real-World Architecture

```
   Edge (CDN, WAF)
        |
   API gateway
        |
   Auth (JWT, OAuth)
        |
   Chat service       Search service      Agent service
        |                  |                    |
        +--- LLM gateway --+--- LLM gateway ----+
        |                                       |
        +--- Vector DB cluster (sharded, replicated)
        +--- Inference pools (large H100, small A10G, batch spot)
        +--- Tool microservices
        +--- Workflow engine (Temporal) for long agents
        +--- Event bus (Kafka)
        +--- Persistence (Postgres, Redis, object store)
        |
   Observability mesh (OpenTelemetry, Prometheus, Tempo, Loki)
        |
   Secrets (Vault)
        |
   CI/CD + IaC
```

## 21.23 Tradeoffs

| Choice | Win | Cost |
|---|---|---|
| Microservices | Independent scale, language choice | Operational complexity |
| Monolith | Simpler ops | Coupled deployments |
| Strong consistency | Predictable | Latency, availability |
| Eventual consistency | Available, fast | Stale reads |
| Sync orchestration | Simple flow | Tight coupling |
| Event-driven | Decoupled, scalable | Eventual, harder debug |

## 21.24 Failure Modes

In any distributed AI system, plan for:
- LLM provider outage.
- Vector DB partition.
- Network slowness between services.
- Cascading retries amplifying load.
- Cold-start delays during scale events.
- Cross-region replication lag.

Defenses: timeouts, retries, circuit breakers, bulkheads, graceful degradation (fallback to a smaller model, fallback to keyword search if vector down, fallback to "service degraded" message).

## 21.25 Interview Questions

- Explain CAP and PACELC. Where does each AI component live?
- When would you use a workflow engine instead of a synchronous loop?
- Walk through retries, idempotency, and backpressure for an embedding pipeline.
- Design a multi-region AI service with regional failover.
- Compare microservices and monolithic architectures for a small AI team.

## 21.26 Hands-on Exercises

1. Map your design from Chapter 1 to distributed components. Identify failure modes per component.
2. Design retry and timeout policies for every external call in that system.
3. Plan graceful degradation paths when each major dependency fails.

## 21.27 Common Mistakes

- No timeouts; piling threads on slow dependencies.
- Retries without idempotency, causing duplicate actions.
- Tight coupling between services; one team blocks another constantly.
- Treating eventual consistency as a bug instead of designing for it.
- Skipping circuit breakers, suffering cascading failures.

## 21.28 Enterprise Best Practices

OpenTelemetry tracing across every service. Documented SLOs and SLIs per component. Failure scenarios documented and rehearsed. Workflow engine for any process with more than a handful of steps or any retry needs. Regular dependency review for unnecessary coupling.
