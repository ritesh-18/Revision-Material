# 07 — Microservices & System Design (25 Questions)

You led a monolith-to-microservices migration (1 → 6 services) at Tibil and architected a multi-tenant SaaS with Kafka fan-out. These cover when and how to decompose, sync vs async communication, sagas, CQRS, resilience patterns (circuit breaker, retry, bulkhead), distributed tracing, idempotency, CAP, and the gotchas senior interviewers care about.

**Sections**
- Part A — Decomposition & boundaries (Q1–Q5)
- Part B — Communication & coordination (Q6–Q12)
- Part C — Resilience patterns (Q13–Q18)
- Part D — Cross-cutting concerns (Q19–Q25)

---

## Part A — Decomposition & Boundaries

### Q1. When does a monolith actually need to be broken into microservices? Walk me through your Tibil migration.

**Answer:**
Most teams reach for microservices too early. The cost is real: more deploys, more network calls, more distributed-systems failure modes, more ops surface. A monolith is the right answer for most teams under ~20 engineers.

**Real signals to decompose:**
1. **Team friction** — multiple teams stepping on each other in the same codebase, deploys blocked by unrelated changes.
2. **Scaling asymmetry** — one feature needs 10x more compute than the rest; scaling the whole monolith for one hot path is wasteful.
3. **Tech-stack divergence** — one component needs Python ML, another needs Rust performance; the monolith forces one language.
4. **Independent release cadence** — different parts have very different change/risk profiles (auth changes are scary monthly; UI changes are weekly).
5. **Bounded contexts emerging** — clear domain boundaries with little shared state.

**Tibil's monolith → 6 services migration sketch:**
The pre-state: one Node monolith handling auth, orders, payments, reports, notifications, and admin in one process. Deploys took 25 min; one team's bug broke other teams; the reports module hammered the DB.

**Approach:**
1. Identify the cleanest seam — for you it was **reports** (read-heavy, scaling independently, few cross-references to the rest).
2. Extract via the **strangler pattern**: stand up a new service, route some traffic via API gateway, keep monolith handling rest, gradually migrate.
3. Move shared DB tables to the new service's own schema. Replace cross-table joins with API calls or denormalized snapshots.
4. Repeat for each bounded context.

**Result:** independent deploys per service, horizontal scaling of hot services, 1K+ req/s sustained because reports no longer competed with payment writes.

**Follow-up 1: What's the strangler-fig pattern?**
Named after the strangler fig vine that gradually replaces its host tree. You stand up a new service for one piece of functionality, route requests to it via a proxy/gateway, leave the old monolith handling everything else. Over time, more functionality migrates; eventually the monolith is hollowed out. No big-bang cutover, low risk.

**Follow-up 2: When is decomposition a mistake?**
When the boundary is wrong. If two "services" need synchronous communication for every operation and share most of their data, you've just turned a function call into a network call with extra latency and failure modes. Test the boundary first by enforcing it in code (separate modules, no shared types, communication via well-defined interfaces). If the monolith works fine with that discipline, you may not need separate processes.

---

### Q2. How do you find the right service boundaries? Talk about DDD bounded contexts.

**Answer:**
**Bounded contexts** is the Domain-Driven Design concept that maps directly to service boundaries. A bounded context is a region of the domain where terms have specific, internally-consistent meanings. "Order" in the sales context means a customer's cart; "order" in the warehouse context means a pick-pack-ship job — same word, different model.

**Finding them:**
1. **Event-storming workshops** — get domain experts in a room, walk through the business in events ("order placed," "payment confirmed," "shipment dispatched"). Clusters of events that share concepts and rules become bounded contexts.
2. **Look for language shifts** — when the same word means different things in different parts of the conversation, you've found a boundary.
3. **Conway's Law** — your service boundaries will eventually mirror your team structure. Reverse-Conway: design service boundaries to match the teams you want.
4. **Data ownership** — each context owns its data. If two services constantly need to join each other's tables, they're probably one context.

**Anti-patterns:**
- **Entity services** — one service per database table ("UserService", "OrderService"). Creates ultra-chatty interactions; no business meaning.
- **Distributed monolith** — services are deployed separately but call each other so much that one going down breaks everything. Worst of both worlds.

**For your SaaS:**
- AuthN/AuthZ (Keycloak integration) — one context.
- Tenant management (onboarding, billing tier) — another.
- Job application workflow — another.
- Notification dispatching — another.

Each owns its data; they communicate via events for state changes.

**Follow-up 1: How small should a microservice be?**
Not a line count. A useful heuristic: "can be rewritten by a small team in 2–4 weeks" or "one engineer can hold the whole thing in their head." If a service spans 20 entities and 80 endpoints, it's probably multiple contexts merged.

**Follow-up 2: What's a "shared kernel" in DDD?**
A small piece of model shared between two contexts (e.g., the `UserId` value type, currency types). Use sparingly — every shared element is coupling. Better than duplicating critical invariants, worse than independence. Versioned carefully like a library.

---

### Q3. Sync vs async communication — when do you use each?

**Answer:**

**Synchronous (HTTP/REST, gRPC):**
- Caller blocks until response.
- Strong tooling: schemas, type safety, debugging clarity.
- Tight coupling: caller knows the callee's address and API; callee being down means caller fails.
- Best for: queries (read data from another service), request-response with low latency requirements.

**Asynchronous (Kafka, RabbitMQ, MQTT):**
- Caller publishes an event; consumers handle it later.
- Loose coupling: publisher doesn't know who consumes.
- Resilience: consumer can be down, messages buffer.
- Best for: state changes that fan out (an order was placed → email service, inventory service, analytics service all react), workflows that span time.

**Hybrid pattern:** sync API for queries + async events for state changes. The user's HTTP call returns immediately ("payment received"); behind the scenes, an event fans out to fulfillment, notifications, analytics.

**Code — synchronous when you need an answer:**
```ts
// User profile service needs current user info; auth service is the source.
const user = await fetch(`http://auth/users/${id}`).then(r => r.json());
```

**Async when you need to notify:**
```ts
// User updated their email; broadcast to interested services
await producer.send({
  topic: 'user.email_changed',
  messages: [{ key: userId, value: JSON.stringify({ userId, oldEmail, newEmail }) }],
});
// Notification service, audit service, billing service all consume independently
```

**Follow-up 1: How do you avoid the "sync call chains" anti-pattern?**
When service A calls B calls C calls D, latency adds up and any one failing breaks everything. Mitigate: (1) reduce chain depth with API composition / aggregation, (2) move dependent state changes to events instead of inline calls, (3) use timeouts and circuit breakers at each hop, (4) cache where possible.

**Follow-up 2: When does async messaging hurt more than help?**
When you need a synchronous answer for a user request. "Did the payment go through?" is sync; the user is waiting. Eventual consistency works for "the analytics will reflect this within a minute," not for "tell me right now if my balance is enough." Mixing the two: respond sync from the system of record, fan out events for derived data.

---

### Q4. REST vs gRPC vs messaging — what fits where?

**Answer:**

**REST (HTTP+JSON):**
- Universal. Every language, every tool, every browser.
- Text-based, debuggable with curl.
- Loose contract — schema discipline is on you (OpenAPI/Swagger).
- Higher overhead per call (HTTP headers, JSON parsing).
- Best for: public APIs, browser ↔ backend, low-frequency cross-service calls.

**gRPC:**
- Binary protocol over HTTP/2.
- Strong typing via Protocol Buffers — schema and code generation are first-class.
- Bidirectional streaming, multiplexed connections.
- Lower latency and bandwidth than REST.
- Best for: internal high-frequency service-to-service calls, polyglot environments with shared `.proto` schemas, streaming.

**Messaging (Kafka/RabbitMQ/etc.):**
- Async, decoupled.
- Durable; survives consumer downtime.
- Best for: state-change propagation, workflows, fan-out.

**Decision matrix:**
| Need | Pick |
|------|------|
| External / public API | REST |
| Internal call, needs answer | gRPC (or REST if no perf issue) |
| Internal call, fire and forget | Messaging |
| Streaming responses | gRPC streaming |
| Event propagation | Messaging |
| Browser ↔ backend | REST (or gRPC-Web) |

**Code — gRPC service definition:**
```protobuf
service Users {
  rpc GetUser (GetUserRequest) returns (User);
  rpc StreamUserEvents (StreamRequest) returns (stream UserEvent);
}

message User {
  string id = 1;
  string email = 2;
  repeated string roles = 3;
}
```

**Follow-up 1: Why does gRPC have lower latency than REST?**
HTTP/2 (multiplexed streams, header compression) + binary Protobuf encoding + persistent connections. A typical internal gRPC call is 1–5ms; REST/JSON on the same path is 5–15ms. The difference matters at high QPS.

**Follow-up 2: When is GraphQL the right answer?**
For composing data across services in a single request — frontend-driven aggregation. Great for product-facing APIs where clients have diverse data needs. Poor fit for high-throughput backend-to-backend (the resolver overhead is a tax) and a learning curve worth weighing.

---

### Q5. What does an API Gateway do?

**Answer:**
An API Gateway is a reverse proxy that sits between clients and backend services. It centralizes cross-cutting concerns so they don't need to be implemented in every service.

**Responsibilities:**
- **Routing** — `/users/*` → users service, `/orders/*` → orders service.
- **Authentication** — verify tokens once at the edge; pass user context downstream.
- **Rate limiting** — per-IP, per-user, per-route.
- **Request/response transformation** — convert public-facing shape to internal.
- **Aggregation** — call multiple backend services and merge the response (the "Backend-for-Frontend" pattern).
- **Caching** — for cacheable GETs.
- **Logging, metrics, tracing** — uniform across all services.
- **Versioning** — `/v1` vs `/v2` routing.
- **CORS, compression, TLS termination.**

**Choices:**
- **Off-the-shelf**: AWS API Gateway, Kong, Tyk, Traefik, Envoy, Nginx with Lua.
- **Build your own** in Node/Go for special-needs cases.

**Anti-pattern:** the "fat gateway" with business logic. The gateway should be thin — routing, auth, rate limiting, transformation. Business logic stays in services.

**Code — minimal Express gateway:**
```js
app.use('/users', auth, rateLimit, proxy('http://users-service:3000'));
app.use('/orders', auth, rateLimit, proxy('http://orders-service:3001'));
```

**Follow-up 1: Should there be one gateway or many?**
"Backend-for-Frontend (BFF) pattern" — one gateway per client type (web BFF, mobile BFF, partner BFF). Each tailored to its consumer's needs. Avoids a one-size-fits-all gateway that becomes a coordination bottleneck. For your SaaS, a single tenant-aware gateway plus a separate webhook receiver is a common split.

**Follow-up 2: How does the gateway hand auth context to downstream services?**
After verifying the JWT once, inject a trusted internal header: `X-User-Id`, `X-Tenant-Id`, `X-Roles`. Downstream services trust this header **only if the network is private** (mTLS, service mesh, VPC). External traffic must be re-authenticated; never trust headers from outside the gateway boundary.

---

## Part B — Communication & Coordination

### Q6. How does service discovery work?

**Answer:**
**Service discovery** is the mechanism by which one service finds the network address of another. In a static world you'd hardcode `users-service:3000`. In a dynamic world (containers come and go, autoscale up and down), you need a registry.

**Patterns:**

**Client-side discovery:**
- Each service queries the registry to get a list of instances of the target service.
- Client load-balances across them.
- Examples: Eureka (Netflix), Consul.
- Pro: lower latency (no extra hop). Con: discovery client baked into every service.

**Server-side discovery:**
- Client calls a fixed address (load balancer or proxy); the proxy knows which instances are alive and routes accordingly.
- Examples: Kubernetes Service, AWS ALB, Nginx with dynamic upstream.
- Pro: clients are dumb; the proxy does the work. Con: extra hop.

**DNS-based discovery:**
- Each service has a stable DNS name; DNS resolves to current instances.
- Kubernetes does this internally — `http://users-service` is resolved to one of the pod IPs.
- Simple and language-agnostic. Caveat: DNS caching can keep stale entries; use short TTLs.

**For your stack:** running on Kubernetes? Use the built-in Service abstraction — DNS-based, no extra components. Running on EC2? An ALB per service or a Consul cluster.

**Follow-up 1: What's a service mesh and how does it relate to discovery?**
A service mesh (Istio, Linkerd) injects a sidecar proxy alongside each service instance. The sidecar handles discovery, TLS, retries, traces — your application code just makes an HTTP call to `localhost:port`. The sidecar figures out where to route it. Heavyweight but powerful for large fleets.

**Follow-up 2: How do you handle a service instance that becomes unhealthy?**
Health checks. The registry / load balancer probes each instance periodically (HTTP GET /health). Failed probes remove the instance from the rotation. Kubernetes does this via readiness probes; the pod doesn't get traffic until /readyz returns 200, and is yanked from rotation when it stops.

---

### Q7. Explain the saga pattern. Orchestration vs choreography?

**Answer:**
A **saga** is a sequence of local transactions across multiple services, where each transaction publishes an event that triggers the next. If any step fails, **compensating transactions** undo previous steps.

Classic example: book a trip = reserve flight + reserve hotel + charge card. No global transaction across three services. If the charge fails, you cancel the hotel and the flight.

**Two coordination styles:**

**Choreography** — services react to events; no central coordinator.
```
Order Service ─publishes─▶ "order_placed"
                              │
            ┌─────────────────┼─────────────────┐
            ▼                 ▼                 ▼
     Inventory Svc      Payment Svc      Notification Svc
     reserves stock     charges card     sends confirmation
            │
       publishes
       "stock_reserved"
            │
            ▼
        Payment Svc
        consumes both and charges
```
- Pro: loose coupling; no central failure point.
- Con: flow is implicit, hard to visualize, easy for one service to forget its part.

**Orchestration** — a central "saga orchestrator" tells services what to do.
```
Order Service ──▶ Saga Orchestrator
                       │
       ┌───────────────┼──────────────┐
       ▼               ▼              ▼
   Reserve Stock   Charge Card    Send Email
       │               │              │
       └─────── reports back to ──────┘
                       │
                   Orchestrator
                   decides next
```
- Pro: explicit, visible workflow; central retry/compensation logic.
- Con: orchestrator is a single point of complexity, can become a god-object.

**Compensation logic** — for each forward action, define a compensating one: "cancel reservation," "refund card." Compensations should be idempotent (may run more than once).

**Frameworks:** Temporal, AWS Step Functions, Camunda, Conductor.

**Follow-up 1: Why not just use a distributed transaction (2PC)?**
Two-phase commit doesn't scale across services with different databases. It requires participants to lock resources during the prepare phase — sub-second is fine, multi-second is catastrophic, and any participant going down stalls everyone. Modern distributed systems avoid 2PC; sagas with compensations are the practical replacement.

**Follow-up 2: What's the hardest part of writing compensations?**
Making them idempotent and handling partial state. If "reserve flight" succeeded but the system crashed before recording it, when you retry the compensation ("cancel flight"), you have to handle "we may or may not have reserved." The trick: every step records its result before being considered done; compensations check and act accordingly.

---

### Q8. How does CQRS apply in microservices? When is it the right pattern?

**Answer:**
**CQRS (Command Query Responsibility Segregation)** separates the model used for **writes** (commands) from the model used for **reads** (queries). They may live in entirely different services, databases, or schemas.

In a microservices context:
- Command service owns the write model and DB.
- Read services own denormalized views, often updated by events from the command service.
- Each can scale independently and use the storage best suited (Postgres for writes, Elasticsearch for search reads, Redis for hot lookups).

**When CQRS shines:**
- Read/write ratio is heavily skewed (1000:1).
- Reads need multiple specialized representations (search, dashboard, mobile summary, analytics).
- The write model is complex (rich domain rules) but reads are simple (just display).
- You're already using event sourcing — CQRS pairs naturally.

**When it's overkill:**
- Simple CRUD with no asymmetry.
- The team isn't ready for eventual consistency between writes and reads.

**Code sketch:**
```
[Write Side]                      [Event Stream]              [Read Side]
   API → OrderService → Postgres ──▶ "order_placed" ──▶ Search Indexer → ES
                                                       Dashboard Builder → Redis
                                                       Analytics → Snowflake
```

The write service is the only one that mutates orders; everything else is built from events.

**Follow-up 1: What's the trade-off of eventual consistency between write and read sides?**
Users see their own writes with a delay. "I just placed an order but it doesn't show in my orders page." Mitigations: (1) read-your-writes — after a write, return optimistic data from the write side or hint the read side; (2) very fast event propagation (sub-second); (3) UI tells the user "your order is being processed" if the read side hasn't caught up.

**Follow-up 2: Can CQRS share a database between command and query?**
Yes — they can be separate models on the same DB, with views or materialized views serving queries. This is "CQRS-lite" — gives you the architectural benefit (separate write and read models) without the eventual-consistency cost. Use until you outgrow it.

---

### Q9. Event sourcing — what is it and when do you use it?

**Answer:**
In event sourcing, you don't store the **current state** of an entity. You store the **stream of events** that produced that state. The current state is the result of replaying all events.

```
                 events                          current state
                                                  (rebuilt)
   ┌──────────────────────────────┐
   │ OrderCreated(o123, user42)   │
   │ ItemAdded(o123, widget)      │ ──▶ replay ──▶ Order { id:o123, user:42,
   │ ItemAdded(o123, gadget)      │                       items:[widget, gadget],
   │ Discount(o123, 10%)          │                       discount:10%,
   │ OrderPlaced(o123)             │                       status:'placed' }
   └──────────────────────────────┘
```

**Benefits:**
- **Full audit trail.** Every state change is preserved.
- **Time travel.** "What did this order look like at 3pm yesterday?" Replay to that point.
- **Multiple read models.** Build whatever views you want from the same event stream.
- **Forensics.** Bugs are debuggable because you can replay exactly what happened.
- **Event streams are first-class** — easy to integrate with downstream systems.

**Costs:**
- **Eventual consistency** between writes and projections.
- **Schema evolution is harder** — old events stay forever; you must handle them on replay even if the code changed.
- **Snapshots** are needed for entities with thousands of events, or replay gets slow.
- **Mental model shift** for the team.

**For your SaaS:** event sourcing the *audit log* and *job application state* could give you free replay and time-travel for compliance. Sourcing every entity is usually overkill.

**Follow-up 1: How do you handle schema changes to events?**
Two rules: (1) old events are immutable — never modify a stored event; (2) when replaying, upcast old shapes to new on the fly. If you can't avoid breaking changes, version your events (`OrderPlacedV2`) and write upgraders. This is one of the harder parts of event sourcing; many teams give up here.

**Follow-up 2: What's the difference between event sourcing and an event-driven architecture?**
Event-driven architecture means services communicate via events. Event sourcing means the events ARE the state. You can have event-driven without event sourcing (services persist state normally but publish events as notifications). The reverse is also true (event sourcing internally, but expose REST APIs).

---

### Q10. Explain the circuit breaker pattern.

**Answer:**
The circuit breaker pattern protects a service from a failing downstream by **failing fast** when the downstream is unhealthy, instead of timing out repeatedly.

**Three states:**

```
   ┌──────────┐  failure threshold      ┌──────┐
   │  CLOSED  │ ──────────────────────▶ │ OPEN │
   │ (normal) │                          │      │
   └────▲─────┘                          └──┬───┘
        │                                   │ after cooldown
        │                                   ▼
        │                          ┌────────────────┐
        │   success after probe    │   HALF-OPEN    │
        └──────────────────────────│   (testing)    │
                                   └────────────────┘
                                          │ failure
                                          ▼
                                     back to OPEN
```

- **Closed** — requests flow normally. Failures are counted.
- **Open** — fail-fast for the cooldown period (e.g., 30s). No traffic to the downstream.
- **Half-open** — after cooldown, allow a few probe requests. If they succeed, close the breaker; if they fail, reopen.

**Why it matters:**
Without a breaker, when a downstream becomes slow, every caller waits on it — threads pile up, memory grows, your service also starts failing. The breaker isolates: one downstream's problems don't drag you down.

**Code — using `opossum` (Node circuit breaker):**
```js
const CircuitBreaker = require('opossum');

const breaker = new CircuitBreaker(
  () => fetch(`http://payments/charge`, { body, signal: AbortSignal.timeout(2000) }),
  {
    timeout: 3000,
    errorThresholdPercentage: 50,           // open if >50% fail
    resetTimeout: 30000,                    // try again after 30s
  }
);

breaker.on('open', () => alert.send('payments breaker opened'));
breaker.fallback(() => ({ status: 'unavailable', queueFor: 'retry' }));

const result = await breaker.fire();
```

**The fallback** is the key — what to do when the breaker is open. Options: return cached data, return a default, queue the request for later, return a clean error.

**Follow-up 1: What happens to in-flight requests when the breaker opens?**
They continue; the breaker doesn't cancel them. New requests fail immediately. This is usually what you want — don't add work-cancellation complexity unless your timeouts make in-flight requests obviously hopeless.

**Follow-up 2: Should the breaker count timeouts as failures?**
Almost always yes — a timed-out request is a failed request from the breaker's perspective. The whole point is to recognize "downstream is slow" early. If you exclude timeouts, the breaker stays closed while every request takes the full timeout duration — defeating the purpose.

---

### Q11. How do retries with exponential backoff work? What's "jitter" and why does it matter?

**Answer:**
When a request fails transiently (timeout, 502, 503), retrying often succeeds. But retrying immediately is bad: it adds load to the already-struggling downstream.

**Exponential backoff:** wait 1s, then 2s, then 4s, then 8s, etc. Each retry doubles the wait.

```
attempt:  1     2     3     4     5
wait:     1s    2s    4s    8s    16s   (then give up)
```

**Jitter:** add randomness to the wait. Without it, if 1000 clients all start at the same time, they all retry at exactly 1s, 2s, 4s — synchronizing onto the downstream like a wave. With jitter, retries spread out:
```
wait = min(MAX, BASE * 2^attempt) * uniform(0, 1)
```
Or "full jitter": `wait = uniform(0, BASE * 2^attempt)`.

**Code:**
```js
async function fetchWithRetry(url, opts, max = 5) {
  let attempt = 0;
  while (true) {
    try {
      const res = await fetch(url, opts);
      if (res.status < 500) return res;     // 4xx is permanent
      throw new Error(`HTTP ${res.status}`);
    } catch (err) {
      attempt++;
      if (attempt >= max) throw err;
      const base = Math.min(30_000, 1000 * 2 ** attempt);
      const wait = Math.random() * base;
      await new Promise(r => setTimeout(r, wait));
    }
  }
}
```

**What NOT to retry:**
- **4xx (except 429)** — permanent client errors. Retry won't fix them.
- **Non-idempotent operations without idempotency keys.** Retrying a `POST /charge` could double-charge.
- **Requests already taking too long** — context likely cancelled.

**Follow-up 1: What's the difference between "retry budget" and unbounded retries?**
A retry budget caps the additional load from retries (e.g., "at most 10% extra requests are retries"). Without a budget, a downstream meltdown causes retries to multiply traffic, making the meltdown worse. Smart clients implement an adaptive budget: if recent success rate is low, fewer retries.

**Follow-up 2: How do you retry safely for `POST` operations?**
Use idempotency keys (Q15 in section 11). The client generates a unique key per logical operation and sends it in a header (`Idempotency-Key`); the server records the key and the operation's result. Retries with the same key get the same response, no duplicate effect. Stripe's API and your payment system at Dhee both use this pattern.

---

### Q12. What's the bulkhead pattern?

**Answer:**
Inspired by ship bulkheads — internal walls that contain flooding to one compartment instead of sinking the ship. In software: isolate resources so a problem in one area doesn't drain everything.

**Common bulkheads:**
- **Connection pools per dependency.** Separate Postgres pool for the orders DB and the analytics DB. If analytics queries pile up, they don't starve orders.
- **Thread pools / async worker pools per workload.** Background jobs run on a different pool than HTTP request handlers.
- **Separate Kubernetes deployments.** Cron jobs in their own deployment so a misbehaving cron doesn't OOM the API pods.
- **Rate limiting per tenant / per consumer.** One noisy tenant can't starve others.

**Without bulkheads:** one slow downstream makes every request wait on the shared pool, exhausting it. Symptoms cascade across the entire service.

**With bulkheads:** one bad path consumes its own pool's resources; other paths keep working.

**Code — separate HTTP agents per upstream:**
```js
const paymentsAgent = new http.Agent({ maxSockets: 50 });
const inventoryAgent = new http.Agent({ maxSockets: 100 });

await fetch('http://payments/charge', { agent: paymentsAgent });
await fetch('http://inventory/check', { agent: inventoryAgent });
```

Now a payments outage tying up its 50 sockets doesn't starve inventory.

**Follow-up 1: What's the cost of over-bulkheading?**
Wasted capacity. If you give each of 20 dependencies its own 50-connection pool, you've reserved 1000 connections, most idle most of the time. Pick bulkhead boundaries that match where you've actually seen contention.

**Follow-up 2: How does the bulkhead pattern relate to the circuit breaker?**
Complementary. Circuit breaker fails fast when downstream is bad; bulkhead ensures that even if it doesn't fail fast, the damage is contained. Real-world resilient services use both.

---

## Part C — Resilience Patterns

### Q13. How do you choose timeouts? Connect, read, request — what's the difference?

**Answer:**
Different timeouts protect against different failure modes.

- **Connect timeout** — time to establish a TCP connection. Should be short (1–2s). A long connect time usually means the target is down or DNS is broken; no point waiting.
- **Read timeout / socket timeout** — time between bytes on an active connection. If the server is responding but slowly, you might want to wait longer.
- **Total request timeout** — wall-clock time for the whole operation. The most important one — caps the entire request.
- **Pool acquire timeout** — time to get a connection from the pool. If the pool is exhausted, fail fast rather than letting requests queue forever.

**Setting them:**
- **Downstream-aware.** If the downstream's p99 is 500ms, your timeout should be ~1s (2x buffer). Anything longer just keeps you waiting after the downstream has effectively failed.
- **Less than your own request timeout.** If your API has a 30s budget, no downstream should be allowed to consume all of it.
- **Cascade-aware.** A's timeout to B should be less than the caller's timeout to A, minus A's own work time.

**Code — composing timeouts:**
```js
const controller = new AbortController();
const timeout = setTimeout(() => controller.abort(), 2000);
try {
  const res = await fetch('http://downstream', { signal: controller.signal });
} finally {
  clearTimeout(timeout);
}
```

**Without timeouts at every layer:** one slow downstream causes a thread/connection to hang forever. Pool exhausts. Whole service becomes unresponsive. Always set timeouts.

**Follow-up 1: How do timeouts interact with retries?**
The retry budget must respect the total deadline. If you have 30s and the first attempt takes 5s before timing out, you can retry with a budget of 25s minus backoff. Naïvely retrying with full timeout each time blows past the user's wait budget.

**Follow-up 2: What's a "deadline" vs a timeout?**
Timeout is local: "this operation should take less than N seconds." Deadline is global: "this entire request must finish by T." Deadlines propagate through call chains — service A's deadline gets passed to B (e.g., via a `Deadline` gRPC metadata or `X-Deadline` header), which trims its own timeouts accordingly. Better than independent timeouts that don't sum.

---

### Q14. How does distributed tracing work and what is OpenTelemetry?

**Answer:**
Distributed tracing follows a single request as it flows through multiple services. Each step is a **span**; spans share a **trace ID** that ties them together.

```
trace_id: abc-123
  ▶ span (api gateway, 250ms)
      ▶ span (auth service, 30ms)
      ▶ span (orders service, 180ms)
          ▶ span (postgres, 50ms)
          ▶ span (inventory service, 100ms)
              ▶ span (postgres, 40ms)
```

Spans capture: trace ID, span ID, parent span ID, name, start/end time, attributes, events.

**W3C trace context** standardizes the HTTP header format (`traceparent: 00-{trace}-{span}-{flags}`), so traces flow across services regardless of vendor.

**OpenTelemetry (OTel)** is the open standard:
- **API** — what your code uses to create spans, set attributes.
- **SDK** — implementations.
- **Collector** — receives data from apps, exports to backends (Jaeger, Tempo, Datadog).

**Code — Node with OTel:**
```js
const { NodeSDK } = require('@opentelemetry/sdk-node');
const { OTLPTraceExporter } = require('@opentelemetry/exporter-trace-otlp-http');
const { getNodeAutoInstrumentations } = require('@opentelemetry/auto-instrumentations-node');

const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter({ url: 'http://otel-collector:4318/v1/traces' }),
  instrumentations: [getNodeAutoInstrumentations()],
});
sdk.start();

// Auto-instrumentation wires HTTP, Postgres, Redis, Kafka clients automatically.
// Custom span when you need it:
const tracer = trace.getTracer('users-service');
await tracer.startActiveSpan('build-dashboard', async span => {
  span.setAttribute('user.id', userId);
  // ... work
  span.end();
});
```

**Follow-up 1: Why not just log everything with the request ID?**
You can, and many teams do (log correlation). But tracing visualizes the parent-child relationships and timing — much faster to spot "this slow request is slow because of this one DB query in this one service." Logs are great for content; traces are great for shape.

**Follow-up 2: What's "sampling" in tracing and why does it matter?**
Capturing every request is expensive (bandwidth, storage). Sampling captures only a fraction. **Head sampling** decides at the start (e.g., 1% random). **Tail sampling** decides at the end based on outcome (e.g., always sample errors and slow requests, 1% of healthy). Tail is more useful but requires buffering until the trace completes — heavier infra.

---

### Q15. What's an idempotency key and why does every payment API use one?

**Answer:**
A non-idempotent operation produces different results when repeated (e.g., "create payment of $100"). If the client retries because of a network blip, the server might process twice — double charge.

**Idempotency key** = a client-generated unique ID per logical operation. The server records the key and the response. On a retry with the same key, the server returns the previously recorded response instead of re-executing.

**Code — typical implementation:**
```js
app.post('/payments', async (req, res) => {
  const key = req.headers['idempotency-key'];
  if (!key) return res.status(400).json({ error: 'idempotency-key required' });

  // Check if we've seen this key
  const existing = await db.idempotency.findUnique({ where: { key } });
  if (existing) {
    // Return the recorded response
    return res.status(existing.status).json(existing.body);
  }

  // Process and record
  const result = await processPayment(req.body);
  await db.idempotency.create({
    data: { key, status: 200, body: result, expiresAt: addDays(now(), 1) },
  });
  return res.json(result);
});
```

**Subtleties:**
- **In-flight handling** — if two requests with the same key arrive at the same time, both check "is it processed yet?" and find no, both proceed. Use a unique constraint on the key + try-catch-retry, or take a lock.
- **TTL** — clean up keys after 24h or so. Idempotency is for retries, not lifetime deduplication.
- **Tie the key to the request body.** A request with key=X and amount=100 then a follow-up with key=X and amount=200 should be flagged as a conflict, not silently return the first.

**Your Dhee Coding Lab project** used this pattern with payment, SMS, and email providers — "zero duplicate transactions under failure scenarios" comes directly from idempotency keys plus retry-safe state machines.

**Follow-up 1: Why doesn't HTTP itself solve this?**
HTTP defines semantics: GET, PUT, DELETE are idempotent by spec; POST and PATCH are not. But "idempotent by spec" requires the server to implement it; HTTP doesn't enforce. Idempotency keys are how you opt POST into idempotency for retries.

**Follow-up 2: What's the difference between idempotent and deterministic?**
Idempotent: same effect when repeated (e.g., "set balance to 100" — running it once vs 5 times leaves balance at 100). Deterministic: same output for same input. Idempotency is about side effects; determinism is about return value. "Set balance to 100" is idempotent; "create user with random ID" is non-deterministic but can be idempotent if a key dedupes the side effect.

---

### Q16. Explain eventual consistency. How do you reason about it?

**Answer:**
**Eventual consistency** = the system will converge to a consistent state given enough time without new updates. Between writes and convergence, different observers may see different values.

This is the default consistency model when:
- Data is replicated across multiple nodes (Mongo replicas, Postgres read replicas).
- State is propagated via events (Kafka → consumers update local views).
- Caches lag behind sources of truth.

**Examples in your stack:**
- A user updates their profile; Postgres commits; the Redis cache is invalidated; another service reading from Kafka updates its denormalized view 200ms later.
- A multi-region setup where a write in EU propagates to US after some lag.

**Reasoning techniques:**
- **Define the convergence guarantee.** "Within 1 second of a write, all readers see it." Or "within 30 seconds." Or "eventually."
- **Bound the staleness.** Use TTLs on caches, set max replication lag alerts.
- **Read-your-writes when needed.** After a user's own write, route their next read to the source or pass a hint.
- **Handle reorders.** Events may arrive out of order across partitions. Design idempotent handlers that don't break under retries or reorders.
- **Use vector clocks or version numbers** when ordering matters.

**Anti-patterns:**
- Assuming reads see writes immediately. Code with `db.write(); db.read()` works in tests but breaks in prod with read replicas.
- Cascading derived state without idempotency. One service's "stale read of source" propagates to derived stores that compound the staleness.

**Follow-up 1: When is strong consistency worth the cost?**
For correctness invariants users care about: account balances, identity, authorization. Pay the latency cost (synchronous replication, distributed locks, primary-only reads) where wrong answers cause real harm. For analytics, search, dashboards — eventual is fine.

**Follow-up 2: What's "read your writes" and how do you implement it?**
A session-level consistency: a user's reads see at least their own previous writes, even if not other people's writes. Implement by: (1) routing the user's read to the primary for N seconds after a write; (2) including a "min version" header from the client so the server waits for replica to catch up; (3) returning the written value optimistically.

---

### Q17. State the CAP theorem and give a real example of trading off.

**Answer:**
**CAP** (Brewer): in the presence of a network **P**artition, a distributed system can guarantee at most one of **C**onsistency (every read sees the latest write) and **A**vailability (every request gets a non-error response).

The catch: P is not optional. Networks partition. So the practical choice is **CP** (sacrifice availability under partition) or **AP** (sacrifice consistency under partition).

**Real systems:**
- **CP (consistency-prioritizing)**: Postgres with synchronous replication, etcd, ZooKeeper, MongoDB with `w: 'majority'`. During a partition, the minority side becomes read-only or unavailable.
- **AP (availability-prioritizing)**: Cassandra, DynamoDB (default), Riak. During a partition, both sides accept writes; reconcile later with last-write-wins or CRDTs.

**Note:** CAP is about *partition tolerance*. In normal operation, you can have both C and A — Postgres on healthy network is consistent and available. The trade-off applies only when the network breaks.

**PACELC** is a useful extension: even when not partitioned, there's a latency-vs-consistency trade-off. Synchronous replication is consistent but slow; async is fast but eventually consistent.

**For your SaaS multi-tenant project:**
- Auth and tenant data → CP. You want a consistent answer to "what's this tenant's plan?"
- Notification fan-out → AP. If one consumer is partitioned, you let it drift and resync; you don't refuse to send notifications.

**Follow-up 1: How do CAP and ACID relate?**
ACID is about single-DB transaction guarantees. CAP is about distributed systems. They overlap conceptually around consistency, but ACID's C (consistency = integrity rules) is different from CAP's C (linearizability across nodes). Best to think of them as orthogonal.

**Follow-up 2: Why is CAP often called misleading?**
Because real systems aren't binary CP or AP — they have nuanced semantics under partition (degraded reads, partial writes, manual reconciliation). And the "you must give up one" framing oversimplifies; you can offer different guarantees to different operations on the same system. CAP is a useful starting point, not the whole picture.

---

### Q18. ACID vs BASE — which one and when?

**Answer:**

**ACID** (Postgres, MySQL, single-node):
- Atomicity, Consistency, Isolation, Durability per transaction.
- Strong guarantees; programmers reason about state as if no one else is touching it.
- Cost: harder to scale horizontally; locking/coordination overhead.

**BASE** (Cassandra, DynamoDB, Couchbase):
- Basically Available — system is available even with failures.
- Soft state — state can change without input due to eventual consistency.
- Eventually consistent.
- Cost: programmers must handle out-of-order updates, conflict resolution, stale reads.

**When to pick which:**
- **ACID** — when correctness requires strong invariants per operation: financial, identity, regulated data, anything where "wrong answer" is unacceptable.
- **BASE** — when scale requires distribution beyond what ACID systems handle, and the data tolerates eventual consistency: analytics, logs, content feeds, large-scale catalogs.

**Hybrid is normal.** Your SaaS likely uses ACID Postgres for tenant data and BASE-style event streams (Kafka) for fan-out. The trick is keeping the boundary clear: Postgres is the source of truth; Kafka is the eventually-consistent broadcast.

**Follow-up 1: Are Mongo, Redis, and Postgres ACID, BASE, or both?**
- Postgres: fully ACID.
- Mongo: ACID for single documents always; multi-document transactions added in 4.0 (replica sets) and 4.2 (sharded). Default behavior outside transactions is closer to ACID-per-doc than BASE.
- Redis: ACID-ish for single commands and MULTI/EXEC blocks within one Redis instance. With replication and persistence config, durability is configurable. Calling it BASE or ACID depends on usage.

**Follow-up 2: How do you "fake" ACID across services?**
You don't, fully. The closest is: (1) outbox pattern (Q24) for "DB write + event publish" atomicity; (2) sagas with compensations for multi-step workflows. Neither is true ACID — both accept eventual consistency. Real cross-service ACID would require distributed transactions (2PC), which doesn't scale.

---

## Part D — Cross-Cutting Concerns

### Q19. What does a service mesh do? When do you actually need one?

**Answer:**
A service mesh (Istio, Linkerd, Consul Connect) adds a sidecar proxy to every service pod. All inter-service traffic flows through these proxies. The mesh control plane configures them to provide:

- **mTLS** — automatic encryption and authentication between services.
- **Traffic management** — canary deploys, traffic splitting, retries, timeouts, circuit breaking.
- **Observability** — uniform metrics, traces, logs without app instrumentation.
- **Policy** — who can call whom, rate limits, fine-grained authz.

**You need a mesh when:**
- 20+ services and growing.
- Strict security requirements (mTLS everywhere, zero-trust networking).
- Need uniform traffic policies that you don't want to duplicate per service.
- Team is comfortable with the operational complexity.

**You don't need a mesh when:**
- < 10 services. Most of what a mesh provides can be done in code or with a simpler library (e.g., a shared HTTP client with retries and circuit breakers).
- Performance-sensitive — each hop adds ~1ms of proxy latency.
- Your team can't afford the ops investment.

**Follow-up 1: What's the difference between Istio and Linkerd?**
Istio is more featureful and powerful (advanced policy, multi-cluster) but heavier (Envoy sidecars, larger control plane). Linkerd is lighter, faster sidecar (Rust), simpler config. For most teams, Linkerd is the saner default; Istio when you genuinely need its breadth.

**Follow-up 2: Can I get most of the value without a mesh?**
Yes, for many teams. Use a shared HTTP client library that bakes in retries, circuit breaking, tracing. Use Kubernetes NetworkPolicies for basic isolation. Use mTLS via a simpler mechanism (e.g., SPIFFE/SPIRE, or cert-manager + envoy as standalone). Saves operational cost; loses unified-config and live-reconfig benefits.

---

### Q20. What are liveness, readiness, and startup probes?

**Answer:**
Kubernetes-style health checks let the orchestrator know what state your pod is in.

- **Liveness probe** — "are you alive?" If this fails, K8s **kills the pod and restarts it**. Use sparingly; restarts mid-request hurt. Best used to detect genuine deadlocks (e.g., event loop stuck).
- **Readiness probe** — "are you ready to accept traffic?" If this fails, K8s **removes the pod from the Service's endpoints** (no traffic routed) but doesn't restart. Use for transient unavailability — startup, draining for shutdown, downstream dependency outages.
- **Startup probe** — "are you done starting up yet?" Runs before liveness/readiness. Use for slow-starting apps to avoid premature liveness kills.

**Common mistake:** making liveness probe equivalent to "DB is reachable." If the DB blips, K8s restarts every pod simultaneously — making the outage worse. Liveness should test the pod itself (event loop, basic API responsiveness), readiness should test the dependencies.

**Code — Node app exposing both:**
```js
app.get('/livez', (req, res) => res.json({ ok: true }));     // pure aliveness

app.get('/readyz', async (req, res) => {
  try {
    await db.$queryRaw`SELECT 1`;
    await redis.ping();
    return res.json({ ok: true });
  } catch (e) {
    return res.status(503).json({ ok: false, error: e.message });
  }
});
```

```yaml
# deployment.yaml
livenessProbe:
  httpGet: { path: /livez, port: 3000 }
  periodSeconds: 10
  failureThreshold: 3
readinessProbe:
  httpGet: { path: /readyz, port: 3000 }
  periodSeconds: 5
  failureThreshold: 1
startupProbe:
  httpGet: { path: /livez, port: 3000 }
  failureThreshold: 30
  periodSeconds: 2
```

**Follow-up 1: How do you handle graceful shutdown with readiness?**
On SIGTERM: (1) set a flag making readiness return 503; (2) wait `terminationGracePeriodSeconds - 5s` so K8s has time to stop sending traffic; (3) call `server.close()` to refuse new connections, wait for in-flight to finish; (4) close DB pool, exit.

**Follow-up 2: What about probes for non-HTTP services (Kafka consumers)?**
Same idea, different signal. Run a tiny HTTP server inside the consumer process exposing /readyz that reports whether the consumer has joined the group, is processing, and lag is within bounds. Even non-HTTP apps in K8s usually expose a minimal health HTTP endpoint.

---

### Q21. How do you rate-limit at scale?

**Answer:**
At a single server, rate-limiting is a counter per user. At scale (multiple replicas), counters must be shared — typically Redis.

**Algorithms** (covered in Q11 of section 05):
- Fixed window — simplest, boundary issue.
- Sliding window log — exact, memory-heavy.
- Sliding window counter — approximate, cheap.
- Token bucket — supports bursts.
- Leaky bucket — smooths traffic.

**At which layer?**
- **API Gateway** — first line of defense, especially for unauthenticated traffic. Stop bad actors at the edge.
- **Per-service** — for protecting downstream systems, applying business-level limits.
- **Per-tenant** — for multi-tenant SaaS; one noisy tenant can't starve others.

**Quotas vs rate limits:**
- Rate limit: "X requests per second."
- Quota: "Y requests per day/month."

Often you want both: 100/second for burst protection, 1M/day as billing tier limit.

**Code — `rate-limiter-flexible` in NestJS:**
```ts
import { RateLimiterRedis } from 'rate-limiter-flexible';

const limiter = new RateLimiterRedis({
  storeClient: redis,
  keyPrefix: 'rl',
  points: 100,            // 100 requests
  duration: 60,           // per 60 seconds
});

@Injectable()
export class RateLimitGuard implements CanActivate {
  async canActivate(ctx: ExecutionContext) {
    const req = ctx.switchToHttp().getRequest();
    const key = req.user?.id ?? req.ip;
    try {
      await limiter.consume(key);
      return true;
    } catch (rejRes) {
      throw new HttpException('Too Many Requests', 429);
    }
  }
}
```

**Response headers** — include `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` so clients know their budget. For 429, include `Retry-After`.

**Follow-up 1: How do you avoid making rate limiting a single point of failure?**
Redis cluster + circuit-breaker on the limiter itself. If Redis is unreachable, fall back to a more permissive in-memory limit (degraded but not catastrophic) or fail-open. Some teams fail-closed for security-sensitive endpoints. The trade-off is service availability vs abuse exposure.

**Follow-up 2: Can rate limiting be distributed-counter-free?**
Yes — "stochastic rate limiting" where each replica enforces a probabilistically scaled limit (replica enforces limit / N for N replicas). Works for high-volume limits where exactness isn't required. No coordination cost. Bad when limits are tight or precision matters.

---

### Q22. How should logging, metrics, and tracing complement each other?

**Answer:**
Three pillars of observability, each answering a different question.

**Logs** — what happened? Verbose, high-cardinality records.
- Best for: detailed forensics, debugging, audit trails.
- Cost: storage, search cost grows with volume.
- Tip: structured (JSON) only, never plain text. Use levels rigorously.

**Metrics** — how often / how much / how long? Pre-aggregated counters, gauges, histograms.
- Best for: dashboards, alerts, capacity planning.
- Cost: cardinality explosion if you tag every metric with user-id-level labels.
- Tip: low-cardinality labels (service, endpoint, status code), not "userId."

**Traces** — where did time go? End-to-end timing of a single request across services.
- Best for: latency analysis, dependency mapping, slow-query attribution.
- Cost: sampling required at scale (Q14).
- Tip: tag spans with high-cardinality stuff that's too expensive for metrics (user ID, tenant ID, trace ID).

**Correlation:** every log and span should carry the trace ID. When investigating a slow request, find the trace, then drill into logs filtered by that trace ID. This is where most observability investment pays off.

**Code — structured logging with trace context:**
```js
import { trace } from '@opentelemetry/api';

function log(level, msg, fields = {}) {
  const span = trace.getActiveSpan();
  const traceId = span?.spanContext().traceId;
  const spanId = span?.spanContext().spanId;
  console.log(JSON.stringify({
    level, msg, time: new Date().toISOString(), traceId, spanId, ...fields,
  }));
}
```

**Follow-up 1: What's the cardinality problem in metrics?**
Each unique combination of label values creates a separate time series in your metrics backend. Tag with `user_id` and you'll have millions of time series — most systems can't handle that and break or charge a fortune. Keep labels low-cardinality (a handful of values each). Use traces or logs for per-user / per-request detail.

**Follow-up 2: Should errors go in logs, metrics, or traces?**
All three. Logs capture the full error context (stack, request body). Metrics count errors by type (rate alerts). Traces show *where* the error occurred in the call chain. Tools like Sentry combine all three for error tracking, but you should still emit structured logs and counters for general observability.

---

### Q23. "Database per service" — what does it mean in practice?

**Answer:**
In microservices, each service owns its data — no other service writes to its DB, and ideally no other service reads from it directly either. Cross-service data access goes through APIs or events.

**Why:**
- **Schema autonomy.** Service A can change its schema without coordinating with everyone using its DB.
- **Tech choice freedom.** A service that needs Mongo can use Mongo; another that needs Postgres can use Postgres.
- **Scaling independence.** A read-heavy service scales its DB independently of write-heavy services.
- **Failure isolation.** One service's DB outage doesn't bring down everyone.

**Implementation:**
- One logical database per service. Could be one Postgres instance with separate schemas/databases for cost reasons, but logically separate.
- Cross-service queries become API calls or event-driven data duplication.
- "View" needs that span services use a denormalized read model fed by events (CQRS).

**What it breaks:**
- **Cross-service joins.** Welcome to N+1 over HTTP — or denormalized snapshots.
- **Cross-service transactions.** No more `BEGIN; UPDATE A; UPDATE B; COMMIT;` — sagas and outbox instead.
- **Reporting / analytics.** Aggregating across services requires ETL into a warehouse.

**Pragmatic compromises:**
- Shared read-replica for analytics queries that span services (read-only).
- "Aggregator" services that own the cross-service view.
- Accept some duplication — UserService keeps `email`, NotificationService also stores `email` to send without API call.

**Follow-up 1: Isn't this a lot of overhead for small services?**
Yes. The cost of database-per-service is real and not worth it for small teams. Pragmatic intermediate: separate schemas within one DB, treated as boundaries; revisit when scale or team size demands it.

**Follow-up 2: How do you handle "I need the user's name to display in this order list" across services?**
Three options: (1) **API call** to UserService at read time — simple, adds latency; (2) **Denormalize** — Orders stores a snapshot of name at order time, accepting eventual updates; (3) **Aggregator service / GraphQL** that calls both. Choose based on read frequency and freshness requirements.

---

### Q24. Walk me through the outbox pattern.

**Answer:**
The **outbox pattern** solves the dual-write problem: how to atomically update your database AND publish an event to Kafka. Without it, you can have:
- Wrote to DB, failed to publish event → downstream out of sync.
- Published event, failed to write to DB → downstream sees an event for state that doesn't exist.

**The pattern:**
1. In one DB transaction: insert into the business table AND insert into an `outbox` table.
2. The DB transaction guarantees both succeed or both fail.
3. A separate process (relay) reads new outbox rows and publishes them to Kafka.
4. Successful publishes mark outbox rows as sent (or delete them).

```
   ┌─────────────────────────────────────────┐
   │  Service                                │
   │  ┌─────────────────────────────────┐    │
   │  │ BEGIN                            │   │
   │  │   INSERT INTO orders ...         │   │
   │  │   INSERT INTO outbox(event,...)  │   │
   │  │ COMMIT (atomic)                  │   │
   │  └─────────────────────────────────┘   │
   └─────────────────────────────────────────┘
                │
                ▼
   ┌─────────────────────────────────────────┐
   │  Outbox Relay (separate process or CDC) │
   │   1. SELECT FROM outbox WHERE sent=false│
   │   2. publish to Kafka                   │
   │   3. UPDATE outbox SET sent=true        │
   └─────────────────────────────────────────┘
                │
                ▼
             Kafka
```

**Two relay strategies:**

**Polling relay** — a worker polls the outbox table periodically. Simple; slight latency.

**Change data capture (CDC)** — Debezium tails the Postgres WAL; outbox inserts are emitted to Kafka automatically. Lower latency; more infra.

**Code — outbox row format:**
```sql
CREATE TABLE outbox (
  id          uuid PRIMARY KEY,
  aggregate   text NOT NULL,        -- 'order', 'user'
  aggregate_id text NOT NULL,
  event_type  text NOT NULL,        -- 'order_placed'
  payload     jsonb NOT NULL,
  occurred_at timestamptz NOT NULL DEFAULT now(),
  sent_at     timestamptz
);

CREATE INDEX outbox_unsent_idx ON outbox (occurred_at) WHERE sent_at IS NULL;
```

Relay query:
```sql
SELECT * FROM outbox WHERE sent_at IS NULL ORDER BY occurred_at LIMIT 100;
-- publish each, then:
UPDATE outbox SET sent_at = now() WHERE id IN ($1, $2, ...);
```

**Follow-up 1: Why not just publish to Kafka first, then DB?**
Because the order matters for failure modes. Publish-first → if DB write fails, you've emitted an event for state that doesn't exist (consumers can't find the order). DB-first via outbox → at-least-once delivery downstream, but the source of truth (DB) is always correct.

**Follow-up 2: How do consumers handle outbox duplicate events?**
Idempotent consumers. Each event has a stable ID (the outbox row's UUID). Consumers store processed event IDs in their own DB and skip duplicates. Or downstream operations are naturally idempotent (UPSERT, set-state).

---

### Q25. What are the top production gotchas in microservices architectures?

**Answer:**

1. **Distributed monolith.** Services so tightly coupled they must deploy together. Tell-tale signs: one service can't be tested or rolled back independently. Fix by refactoring boundaries — usually means merging too-small services.

2. **Sync-call chains.** Service A → B → C → D adds up latency and creates fragile dependency chains. Refactor to events or fan-out at edge.

3. **No idempotency on critical writes.** Retries cause duplicate charges, duplicate orders. Idempotency keys from day one.

4. **Missing or weak observability.** Without traces, debugging across services is guesswork. Without correlation IDs in logs, you can't follow a request.

5. **Health checks that test downstream.** Causes cascading restarts. Liveness should be local; readiness can test deps.

6. **No timeouts.** One slow downstream cascades through the system. Timeouts at every external call.

7. **Distributed transactions via 2PC.** Doesn't scale, hard to debug. Use sagas with compensations or the outbox pattern.

8. **Shared databases.** Two services writing to the same DB = ongoing coordination tax. Worse: undocumented dependencies.

9. **No DLQ for async workflows.** One bad message blocks a partition.

10. **No version policy for events.** Producers add fields, consumers break. Use schema registry or strict versioning rules.

11. **Inconsistent error formats across services.** Clients can't handle errors uniformly. Standardize error shape via gateway/contract.

12. **Forgetting cardinality limits in metrics.** Tagging metrics with user_id explodes the metric store. Keep labels low-cardinality.

13. **Auto-scaling without bounded queues.** Spike causes scale-up, scale-up causes more queries to slow downstream, slow downstream causes more pods to scale up — feedback loop. Combine auto-scaling with circuit breakers and shed.

14. **No graceful shutdown.** SIGTERM kills pods mid-request. Implement proper drain with readiness flip + server.close().

15. **Lost context on retries / async hops.** Request ID, user ID, deadline must propagate through events and retries.

**Follow-up 1: What's the single most impactful thing a small team can do for microservice resilience?**
Pick one observability standard (OpenTelemetry, trace IDs in logs) and make it mandatory. The dollar-for-dollar return on tracing is enormous: most production debugging takes 10x longer without it.

**Follow-up 2: If you had to defend a "we don't need microservices" position, what would you say?**
"At our scale, the cost of distributed-systems complexity (latency, failure modes, ops surface) outweighs the benefits (team autonomy, independent scaling). A well-modularized monolith with clear boundaries deploys faster, debugs easier, and avoids the entire class of bugs that come from network calls. We should reach for microservices when monolith pain is real and concrete — not as a default."

---

*End of section 07. Next: Multi-Tenant SaaS (15 questions).*
