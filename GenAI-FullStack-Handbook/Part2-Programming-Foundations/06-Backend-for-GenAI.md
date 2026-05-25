# Chapter 6 — Backend for GenAI

## 6.1 Concept Explanation

A GenAI backend is a distributed system whose hottest dependency happens to be an LLM. Everything else — auth, rate limiting, queueing, retries, persistence, observability — is conventional backend engineering. The novelty is the *shape* of the LLM call: long-tailed latency, streaming, expensive failures, non-deterministic outputs. The backend's job is to absorb that novelty and present a stable contract to clients.

## 6.2 The Anatomy of a GenAI Backend

```
   Client
     |
   API Gateway / Edge
     |
   Authentication & Rate Limiting
     |
   Business Logic Service
     |
     +--> Retrieval (vector DB)
     +--> LLM Inference (provider or self-hosted)
     +--> Tool services (internal APIs)
     +--> Persistence (Postgres, object store)
     +--> Event bus (Kafka, NATS)
     +--> Observability (traces, metrics, logs)
```

Each layer is independently scalable, replaceable, and monitorable. The discipline of separating these concerns is what distinguishes a maintainable AI backend from a Jupyter-notebook-turned-deployment.

## 6.3 FastAPI in Production — Beyond Hello World

FastAPI dominates Python AI backends, covered in Chapter 5. Production-grade usage involves:
- Dependency injection for shared resources (DB pool, LLM client, cache).
- Background tasks for fire-and-forget work (logging, analytics).
- Middleware for tracing, authentication, and request logging.
- Lifespan events for startup and shutdown (warm caches, drain in-flight requests).
- Graceful handling of disconnect mid-stream.

The framework is small; the discipline around it is what makes the system reliable.

## 6.4 Node.js Overview

Node.js is the second major backend for GenAI, dominant in JavaScript-first organizations and in Next.js full-stack apps where the same language serves frontend and backend. Its event-driven I/O model maps naturally to streaming LLM responses.

Strengths: shared language with the frontend, fast cold starts, a huge ecosystem, first-class support from most LLM SDKs (OpenAI, Anthropic publish official Node libraries). Weaknesses: weaker ecosystem for scientific computing, weaker support in some Python-first tools (LangChain has a JS port but Python remains primary).

For pure LLM proxying, Node.js is competitive. For anything involving local model loading, embedding generation, or heavy data preprocessing, Python is still preferred.

## 6.5 NestJS Overview

NestJS is a TypeScript framework that imposes structure (modules, controllers, providers) on Node.js backends. It is to Node what Spring is to Java — opinionated, batteries-included, enterprise-friendly.

When to use it: large teams, complex domains, the value of structure exceeds the cost of ceremony. When to skip it: small teams shipping fast, where structure is overhead.

NestJS pairs naturally with TypeORM or Prisma for data access and with the official OpenAI/Anthropic SDKs for LLM calls. It supports SSE and WebSocket out of the box.

## 6.6 API Gateways

An API gateway sits in front of your backend services and handles concerns that should not live in every microservice: authentication, rate limiting, request routing, TLS termination, logging, and increasingly, prompt routing and LLM-specific concerns.

Common choices:
- **Kong, Tyk, Apigee** — general-purpose API gateways.
- **Envoy** — high-performance proxy used inside service meshes.
- **AWS API Gateway, GCP API Gateway, Azure APIM** — managed cloud options.
- **LiteLLM proxy, Portkey, Helicone** — LLM-specific gateways that add model routing, cost tracking, caching, and provider failover.

The trend is toward LLM-aware gateways because plain HTTP gateways do not understand tokens, models, or prompts. An AI gateway can re-route on model failure, enforce per-team token budgets, and standardize observability across providers.

## 6.7 REST vs gRPC vs WebSocket vs SSE

| Protocol | Best for | Avoid for |
|---|---|---|
| REST | External APIs, broad client support | Streaming, bidirectional |
| gRPC | Internal service-to-service, structured contracts, high throughput | Browser clients (limited support) |
| SSE | Streaming model output to browsers | Bidirectional flows |
| WebSocket | Bidirectional realtime (voice, agent interrupts) | Simple request-response |

In practice: REST or SSE for external client APIs, gRPC for internal microservice communication, WebSocket where bidirectionality is mandatory. Mixing is fine and common.

## 6.8 Authentication for AI Apps

Three identity layers usually coexist:
- **End user** — the human using the chat interface.
- **Application** — the workload calling the backend on behalf of the user.
- **Service** — internal services calling each other.

JWT is the typical user token, OAuth2 the typical authorization flow, mTLS or short-lived service tokens for internal service-to-service.

A pitfall specific to agents: when an agent calls a tool, *whose* permissions apply? The user's, not the service's. Forgetting this is how an agent ends up reading someone else's files. Pass the user identity through every tool call and enforce it at the tool side.

## 6.9 JWT and OAuth in Practice

JWT is a signed token containing claims (user id, roles, expiry). Stateless, easy to verify, but cannot be revoked before expiry without a centralized revocation list — defeating the stateless property. Use short expiries and refresh tokens.

OAuth2 is the flow for third-party authorization — "log in with Google," "let this app access your Notion." The typical AI-app pattern is OAuth for user identity (delegating to an identity provider like Auth0, Okta, Cognito), JWT as the session token, scoped tool permissions per user, refresh tokens stored server-side.

## 6.10 Rate Limiting

LLM calls are expensive and slow. Without rate limiting, one user with a script can drain a token budget meant for a thousand.

Standard algorithms:
- **Token bucket.** Allows bursts up to a maximum, refills at a steady rate. Used for "X requests per minute" semantics.
- **Sliding window.** Counts requests in a moving window. More precise, slightly more storage.
- **Concurrent in-flight cap.** Limits how many simultaneous requests one client can have open. Critical for streaming endpoints where a client can hold connections for minutes.

For GenAI specifically, rate limit on *tokens* (input + output), not just requests, since a single request can be 100,000 tokens. Many providers expose token-per-minute and request-per-minute limits separately; mirror that pattern in your backend.

## 6.11 Distributed Tracing

A single user request might touch eight services. Distributed tracing assigns a trace ID at the entry point and propagates it across all downstream calls so you can reconstruct the full path.

OpenTelemetry is the standard. Backends include Jaeger, Tempo, Honeycomb, Datadog. Chapter 19 covers observability in depth.

GenAI-specific trace attributes worth capturing: model name and version, input token count, output token count, prompt template ID, retrieval source IDs, tool call sequence, cache hit/miss, cost in USD. These let you trace cost and quality regressions to specific code paths.

## 6.12 Real-World Use Cases

A representative production GenAI backend:
- REST endpoint for user-facing chat (SSE streaming).
- WebSocket endpoint for voice and agent interrupts.
- gRPC internal API for tool services.
- Background workers for ingestion, embedding generation, eval runs.
- A scheduled job system for periodic model evaluations and drift detection.

## 6.13 Production Architecture

```
   Internet
     |
   CDN / WAF
     |
   API Gateway (auth, rate limit, routing)
     |
   +--> Chat Service (FastAPI, SSE)
   +--> Search Service (FastAPI)
   +--> Agent Service (LangGraph or custom)
     |
   Service Mesh (mTLS, retries, observability)
     |
   +--> Vector DB
   +--> LLM Gateway (LiteLLM/Portkey) -> multiple LLM providers
   +--> Tool services (Python, Go, Node)
   +--> Postgres / Redis / Object store
   +--> Event bus (Kafka)
   +--> Observability backend
```

## 6.14 Alternatives by Stack

| Stack | Strengths | Weaknesses |
|---|---|---|
| Python + FastAPI | Best ecosystem for AI, fast iteration | GIL, ops overhead |
| Node.js + Express/NestJS | Shared with frontend, fast cold start | Weaker AI tooling |
| Go | Performance, concurrency, small binaries | Weaker AI SDK coverage |
| Rust | Maximum performance, safety | Smaller AI ecosystem, longer dev time |
| Java/Kotlin + Spring | Enterprise integrations | Verbose, slow startup |

Most teams pick Python for the AI-touching services and another stack (Go, Java) for the high-throughput infrastructure around them.

## 6.15 Scaling Challenges

- **Long-tail latency.** LLM responses can take 30 seconds. Standard 30-second HTTP timeouts at every layer (load balancer, gateway, ingress) will silently kill streams. Audit every timeout in the stack.
- **Connection saturation.** Streaming endpoints hold connections. Plan for 10× the connections of a non-streaming service.
- **Bursty cost.** One viral feature can 10× the LLM bill overnight. Per-customer budgets and circuit breakers prevent this.

## 6.16 Security Concerns

- Prompt injection at the API boundary — sanitize where you can, but assume injection will happen and design downstream tools to be safe even with malicious model outputs.
- Tool call permission scoping (covered above).
- Logging discipline — prompts and completions can contain PII, secrets, source code, customer data. Redact at the logging layer, not in application code.
- Per-tenant isolation — multi-tenant chat apps must guarantee that one user's context never leaks into another's prompt. Audit retrieval and session code with this lens.

## 6.17 Deployment Guide

Standard pattern: containerize each service, deploy to Kubernetes, expose via Ingress, mesh internal traffic with Istio or Linkerd. Use Helm or Kustomize for manifest management. Roll out via blue-green or canary. Chapter 15 details this.

## 6.18 Monitoring Strategy

Trace every request end to end. Alert on p99 latency, p99 input tokens, error rate, cache hit rate, retry rate, queue depth. Build a dashboard per service that fits on one screen — if you cannot see the system at a glance, you will not notice it failing.

## 6.19 Cost Optimization

Cache aggressively. Route by difficulty. Compress prompts. Truncate retrieval to what fits in budget. Move long-context workloads to providers with better long-context pricing. Profile the top 10 most expensive endpoints monthly; you will be surprised which ones drift up.

## 6.20 Interview Questions

- Why use SSE over WebSocket for chat?
- How do you rate-limit by tokens rather than requests?
- Walk through how an LLM gateway differs from an API gateway.
- Where do you enforce user permissions in an agent's tool call?
- Design the backend for a chat product with 100k DAU.

## 6.21 Hands-on Exercises

1. Map a request through your previous design from Chapter 1 across the layered architecture in 6.13.
2. For each layer, list the timeout it sets and confirm none truncates a 30-second LLM stream.
3. Design a tenant-isolation strategy for retrieval where two users in the same org share an index but two users in different orgs do not.

## 6.22 Common Mistakes

- Layer-cake timeouts that silently break long requests.
- Forgetting per-token rate limits.
- Letting prompt injection reach a destructive tool unchecked.
- Treating the LLM as a sync function rather than a streaming, cancellable, occasionally-failing dependency.

## 6.23 Enterprise Best Practices

Adopt an LLM gateway from day one — even if it is your own. Standardize on OpenTelemetry. Treat prompts and tool definitions as code, versioned in git. Build a single shared "AI client" library used across services so all teams pick up improvements to retry, cache, and observability. Run quarterly cost reviews by endpoint.
