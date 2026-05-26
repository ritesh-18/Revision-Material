# Chapter 23 — Resilience Patterns

## 23.1 Concept Explanation

Resilience patterns are reusable techniques for building systems that survive failure. They are the toolkit SREs apply to the failure modes from Chapter 22. Each pattern is well-understood, has known implementations, and trades off in specific ways.

The patterns in this chapter are language- and tool-agnostic. They appear in resilience libraries (resilience4j, Polly, gobreaker), service meshes, API gateways, and application code. Knowing the patterns by name lets you reach for the right one quickly.

## 23.2 Timeouts

The simplest and most important pattern: do not wait forever.

Every external call must have a timeout. Without it, one slow dependency hangs threads indefinitely, exhausts resources, and cascades.

Setting timeouts:
- **Connect timeout** — how long to establish a connection (typically 1-5 seconds).
- **Read timeout** — how long for the response after connecting (depends on operation).
- **Total timeout** — sum, including retries (often equal to the user's patience, ~30 seconds for API calls).

A common mistake: defaults that are too high. The HTTP client's 30-second default is the wrong default for most internal calls.

## 23.3 Retries

When an operation fails transiently, retry. When it fails systematically, do not.

Distinguishing transient from systematic:
- 5xx errors — usually retriable.
- 4xx errors — usually not (you sent bad input).
- Timeouts — retriable.
- DNS failures — retriable.
- Auth failures — usually not.

Retry policy:
- **Limited count** — typically 3 attempts total.
- **Exponential backoff** — wait 1s, 2s, 4s.
- **Jitter** — randomize to avoid thundering herd.
- **Retry budget** — global cap on retry rate.

Crucially, retries must be **idempotent-safe** (Chapter 21). Otherwise duplicates.

## 23.4 Circuit Breakers

A circuit breaker stops calling a failing service. Three states:

- **Closed** — calls flow normally.
- **Open** — calls fail immediately, no actual call made.
- **Half-open** — periodically test if service has recovered.

Trips when failures (errors, timeouts) exceed a threshold. Resets after a cool-down period.

Why it matters: when a dependency fails, hammering it with retries makes things worse. The circuit breaker stops the bleeding for both the dependency (gives it time to recover) and the caller (fails fast instead of slowly).

Common implementations:
- **resilience4j** (Java)
- **Polly** (.NET)
- **gobreaker** (Go)
- **opossum** (Node.js)
- **Istio, Envoy** (service mesh layer)

## 23.5 Bulkheads

Named after ship compartments, bulkheads isolate resources so one failure does not flood the system.

Examples:
- Separate thread pools for each downstream dependency.
- Separate connection pools.
- Separate microservices for unrelated workflows.
- Separate Kubernetes namespaces or clusters.

Effect: when one dependency fails, threads waiting on it consume only their pool. Other dependencies continue working.

The cost: more resources upfront (multiple pools, multiple instances). Worth it for critical paths.

## 23.6 Load Shedding

When the system is overloaded, shed work rather than fail catastrophically.

Strategies:
- **Reject low-priority requests first.** Free-tier users before paid; analytics queries before checkouts.
- **Return cached or default responses.**
- **Move to degraded mode.**

The principle: graceful failure beats catastrophic failure.

## 23.7 Rate Limiting

Limit how many requests per unit time a caller can make.

Algorithms:
- **Token bucket** — refills at a rate; capped capacity.
- **Sliding window** — count in moving window.
- **Fixed window** — count in fixed buckets.
- **Leaky bucket** — smooth bursts.

Where to limit:
- At the edge (gateway).
- Per user/tenant.
- Per endpoint.
- Globally.

For API services, rate limits are not optional. Without them, one user with a script can drain your capacity.

## 23.8 Backpressure

When a consumer cannot keep up, signal the producer to slow down.

Mechanisms:
- HTTP 429 with Retry-After.
- TCP window size (built into TCP).
- HTTP/2 stream credits.
- Bounded queues that reject when full.
- Reactive streams APIs.

Backpressure is the mechanism that makes load shedding work. Without it, the system buffers until it dies.

## 23.9 Graceful Degradation

When something fails, reduce functionality rather than fail completely.

Patterns:
- **Cached fallbacks** — serve stale data when source is down.
- **Default responses** — generic content when personalization is unavailable.
- **Feature flags** — turn off non-critical features.
- **Read-only mode** — accept reads, reject writes during DB issues.
- **Asynchronous processing** — accept the request now, process later.

Graceful degradation is what makes user-perceived availability much higher than dependency availability.

## 23.10 Health Checks

The signal that determines if traffic should be routed to an instance.

Types:
- **Liveness** — is it alive? Failure → restart.
- **Readiness** — is it ready for traffic? Failure → remove from pool.
- **Startup** — is it done initializing? Used before readiness applies.
- **Deep health** — exercises dependencies. Use carefully (cascading failures).

The classic mistake: making the deep health check too aggressive. When a downstream is unhealthy, every instance fails health checks, all get removed from rotation, total outage.

Better: shallow checks for routing, deep checks for monitoring.

## 23.11 Fallbacks

When primary fails, fall back to secondary.

Examples:
- Primary DB → read replica (read-only).
- Primary cache → secondary cache.
- Primary search → simpler keyword search.
- Primary recommendation → "popular items."

Fallbacks must be **tested**. Untested fallbacks usually do not work when you need them.

## 23.12 Idempotency Keys

Already covered in Chapter 21, but worth restating in resilience context.

A client generates a unique key per logical operation. The server stores keys; duplicates are detected and the previous result returned.

This makes retries safe. Pay $100 once, even if the client retried because the response was lost.

## 23.13 The Saga Pattern

For multi-step transactions across services, use sagas:
- Each step is a transaction.
- Each step has a compensating action.
- On failure, run compensations to undo completed steps.

Sagas replace distributed transactions in microservice architectures. More work to implement; far more reliable than two-phase commit.

## 23.14 Eventual Consistency Patterns

When you cannot have strong consistency, use patterns that work with eventual:
- **CRDTs** — data types that converge automatically.
- **Compensating events.**
- **Eventual repair jobs** that reconcile drift.
- **Idempotent processing.**

These let systems be available during partitions without losing data.

## 23.15 Outbox Pattern

A common eventual consistency pattern. To reliably emit events from a service:
1. Write event to an "outbox" table in the same DB transaction as the business write.
2. A separate process polls the outbox and publishes to the event bus.
3. After publishing, mark as published.

This avoids the dual-write problem (DB write + Kafka write that can fail independently).

## 23.16 Real-World Use Cases

- A team added circuit breakers in front of every external call. Cascading failures dropped to near zero.
- A SaaS added a "degraded mode" feature flag. During a database incident, they served from cache; users barely noticed.
- A retailer added idempotency keys for orders. Lost responses no longer caused duplicate orders.

## 23.17 Production Architecture

```
   Application request
        |
   Rate limit check (token bucket)
        |
   If degraded mode flag set, return cached response
        |
   Call dependency through:
   +-- Bulkhead (own thread pool)
   +-- Circuit breaker (open if recent failures)
   +-- Timeout (e.g., 500ms)
   +-- Retry (with backoff and jitter, up to 3)
        |
   On failure, fallback (cached, default, degraded)
        |
   Return response
        |
   Observability emitted
```

Service mesh implements many of these uniformly. Application code can also implement them with libraries.

## 23.18 Tradeoffs

| Pattern | Win | Cost |
|---|---|---|
| Timeouts | Bounded waiting | May truncate legitimate slow responses |
| Retries | Recover from transients | Amplify load if not careful |
| Circuit breaker | Fast fail | Adds complexity |
| Bulkheads | Isolation | More resources |
| Load shedding | Survive overload | Some users see failures |
| Graceful degradation | Better perceived availability | Engineering work |
| Idempotency | Safe retries | Storage for keys |

## 23.19 Scaling Challenges

- Tuning parameters at scale (timeouts, retry counts, circuit breaker thresholds).
- Coordinating policies across many services.
- Fallback testing — game days are required.
- Observing pattern effectiveness (circuit breaker openings, retry rate).

## 23.20 Security

- Rate limits also defend against abuse and DoS.
- Idempotency keys must not be predictable (security risk).
- Fallback content must respect access control.

## 23.21 Deployment Guide

Implementing resilience patterns:
1. Audit all external calls. Add timeouts.
2. Add retries (with backoff, jitter, idempotency).
3. Add circuit breakers on critical paths.
4. Implement bulkheads via thread pool separation.
5. Define degraded modes per critical service.
6. Test fallbacks in game days.

## 23.22 Monitoring Strategy

- Timeout rate.
- Retry rate.
- Circuit breaker state transitions.
- Fallback usage rate.
- Rate limit rejections.
- Bulkhead utilization.

## 23.23 Cost Optimization

Resilience patterns reduce incident cost. Track:
- Cost avoided from prevented cascading failures.
- Reduced on-call burden.
- Better SLO compliance.

## 23.24 Interview Questions

- *Explain circuit breakers.*
- *When would you use a bulkhead?*
- *How do you safely retry?*
- *Walk through graceful degradation for a service.*
- *What is the outbox pattern?*

## 23.25 Hands-on Exercises

1. For one external call in your service, design a complete resilience policy (timeout, retry, circuit breaker, fallback).
2. Run a game day where you fail a dependency and watch your patterns behave.
3. Identify three places where you could add graceful degradation.

## 23.26 Common Mistakes

- No timeouts.
- Retries without backoff (storm).
- Circuit breakers tuned wrong (always open or never open).
- Untested fallbacks (do not work when needed).
- Fallback that calls the same broken dependency.

## 23.27 Enterprise Best Practices

Standardize resilience patterns via a library or service mesh. Default timeouts in templates. Production readiness reviews check for these patterns. Game days exercise fallbacks. Patterns observable so you know they are working.
