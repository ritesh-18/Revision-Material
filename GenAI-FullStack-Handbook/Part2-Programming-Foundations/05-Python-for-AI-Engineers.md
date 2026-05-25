# Chapter 5 — Python for AI Engineers

## 5.1 Concept Explanation

Python is the lingua franca of GenAI not because it is fast — it isn't — but because it sits at the intersection of three communities: scientific computing (NumPy, SciPy), machine learning (PyTorch, TensorFlow, JAX), and production web services (FastAPI, asyncio). The result is a single language that lets you load a model, run inference, and expose a streaming HTTP endpoint without leaving your editor.

Hot paths in GenAI are not written in Python — they are in CUDA, C++, and Rust under the hood. Python is the orchestration layer. The skill is knowing where Python's overhead matters and where it does not.

## 5.2 Async Python — Why It Matters Here

GenAI applications spend most of their time *waiting* — waiting for the LLM to produce the next token, waiting for the vector database to return results, waiting for an external API to respond. Synchronous code blocks during these waits, idling a thread per request. At 1,000 concurrent users, that is 1,000 idle threads.

Async Python lets a single thread juggle many in-flight requests. While one request is waiting on the LLM, the runtime schedules another that has work to do. Throughput per server goes up by orders of magnitude with no extra hardware.

The mental model: think of your code as a kitchen. Synchronous Python is a chef who stands and stares at the oven for ten minutes. Asynchronous Python is a chef who puts a pot on the stove, starts chopping vegetables for the next dish, returns to stir when the timer dings, and never stops moving. The kitchen size — your thread count — is the same; the throughput is multiples higher.

## 5.3 The asyncio Foundation

The standard library's `asyncio` module is the event loop that all async Python sits on top of. Coroutines (async functions) yield control whenever they hit an "await" point, returning control to the event loop, which then picks the next ready task.

Two ideas matter operationally:
- **Cooperative scheduling.** Async tasks do not preempt each other. A CPU-bound function inside an async handler blocks every other request on that worker. The fix is offloading CPU work to a thread pool or process pool.
- **Cancellation propagation.** When a client disconnects, you want the in-flight LLM call to be cancelled to free GPU time. Async makes this clean; async-unaware libraries do not.

## 5.4 FastAPI — The Default Web Framework

FastAPI is the de facto Python framework for GenAI APIs because it is async-first, ships with Pydantic for type-validated request bodies, and produces OpenAPI documentation automatically. Every team you join will use it, will be migrating to it, or will explain why they did not.

Why it dominates over Flask: Flask is synchronous by default. Why over Django: Django is great for CRUD apps but heavy for thin LLM proxies. FastAPI is roughly as lightweight as Flask, async like Node, and type-safe like a real backend framework. For this niche, it is hard to beat.

## 5.5 Pydantic — Schema as Contract

Pydantic is the data validation library that powers FastAPI's request and response handling. You declare your schemas as Python classes with type annotations, and Pydantic enforces them at runtime — rejecting malformed inputs, coercing types where reasonable, and serializing outputs.

For GenAI, Pydantic plays a second role: defining the shape of structured outputs you want from the LLM. Most LLM SDKs now accept a Pydantic schema and instruct the model to produce JSON that conforms. This collapses a class of bugs (the model returning slightly malformed JSON) into a class of retries (Pydantic rejects, you re-prompt).

## 5.6 Multiprocessing, Threading, and asyncio — Picking the Right Tool

Python has three concurrency primitives and they solve different problems:

| Tool | Best for | Avoid for |
|---|---|---|
| asyncio | I/O-bound workloads (network, disk, LLM API) | Pure CPU work |
| Threading | Mixed I/O with blocking libraries | CPU-bound (GIL) |
| Multiprocessing | CPU-bound work, isolation, parallel inference | Lightweight tasks (process startup cost) |

Common pattern in GenAI servers: asyncio in the request handler, a thread pool for synchronous library calls, a process pool for any heavy CPU preprocessing, and the GPU model itself living in its own server process accessed via inter-process communication.

The Global Interpreter Lock (GIL) is the reason threading does not parallelize CPU work in CPython. PEP 703 is the proposal to make the GIL optional, currently rolling out in 3.13. It will not fully replace multiprocessing soon, but it matters for the medium-term roadmap.

## 5.7 API Design for LLM Endpoints

Three patterns dominate:

**Request-response.** Client sends a prompt, server returns the full completion. Simple, but high-latency feels broken for chat (multi-second blank screens).

**Server-sent events (SSE) streaming.** Server pushes tokens as they generate. This is the default for chat UX. The user sees text appearing in real time, perceived latency drops to time-to-first-token, and you can cancel mid-stream when the user closes the tab.

**WebSocket.** Bidirectional, persistent connection. Used for voice, multi-turn agent loops with intermediate updates, and any flow where the client also needs to send messages mid-generation.

For most LLM apps, SSE is the right answer. WebSocket adds complexity that is often unnecessary.

## 5.8 Streaming Responses — The Implementation Mental Model

In a streaming endpoint, the handler is a generator (or async generator). The framework iterates over the generator and pushes each chunk to the client as it appears. FastAPI handles the HTTP-level details; your code yields tokens.

Two practical concerns:
- **Backpressure.** If the client is slow to read, the queue fills. Bound your buffers so a slow client cannot exhaust server memory.
- **Cleanup on disconnect.** When the client disconnects mid-stream, the generator should be cancelled and the LLM call aborted. Make sure your inference client supports cancellation; not all do.

## 5.9 WebSocket Basics for AI Apps

WebSocket adds bidirectional messaging. The server can push tokens; the client can send mid-stream interruptions ("stop", "switch topic"). For voice AI, where audio chunks flow both ways, WebSocket is mandatory.

The cost: connections are stateful. Load balancers must support sticky sessions or use a stateless message broker (Redis pub/sub) to forward events between worker instances. Auto-scaling is harder because you cannot rebalance an established connection without breaking it.

## 5.10 Caching with Redis

Redis is the cache of choice because it is fast, supports rich data structures, and ships with pub/sub for cross-instance messaging.

In GenAI apps, Redis is used for:
- **Prompt-response caching.** Hash the prompt, return the cached completion if recent. Saves money on repeated questions.
- **Semantic caching.** Embed the prompt, look up similar embeddings in a vector store, return the cached response if similarity is high. Saves money on paraphrased questions.
- **Rate limiting.** Token bucket per user, stored in Redis, decremented on each request.
- **Session storage.** Conversation history for chat applications.
- **Distributed locks.** Coordinating background jobs across workers.

The semantic cache is the high-leverage one. Most chatbot queries cluster around a small number of common questions; serving 60% of them from cache halves the LLM bill.

## 5.11 Real-World Use Cases

A typical FastAPI-based GenAI server handles:
- Authentication via JWT.
- Rate limiting via Redis.
- Request validation via Pydantic.
- Streaming responses to the client via SSE.
- Streaming requests to the LLM via async HTTP.
- Caching at both raw and semantic levels.
- Background logging to an observability backend.
- Tool calls dispatched to internal microservices.
- Graceful shutdown that drains in-flight requests.

None of this requires exotic Python. It requires discipline about which features to use and which to leave alone.

## 5.12 Production Architecture

```
   Client (browser, mobile)
        |
        | SSE / WebSocket
        v
   FastAPI server (async)
        |
        +--> Redis (cache, rate limit, sessions)
        +--> Vector DB (retrieval)
        +--> LLM provider (streaming HTTPS)
        +--> Tool microservices (HTTP)
        +--> Observability backend (async push)
```

Multiple FastAPI workers behind a reverse proxy (typically nginx or a cloud load balancer). Each worker is a separate process. Within a worker, asyncio handles many concurrent requests on a single thread.

## 5.13 Tradeoffs

- **Async everywhere vs only at the edges.** Pure async is cleanest. But many libraries (some database drivers, some ML libraries) are sync only. Mixing requires care.
- **Pydantic v1 vs v2.** v2 is much faster and is the default in new projects. v1 still exists in legacy code; migration is a real effort.
- **FastAPI vs lightweight alternatives (Starlette, Litestar).** FastAPI is opinionated and feature-rich. Litestar is more performant in some benchmarks. For most teams, FastAPI's ecosystem outweighs the performance margin.

## 5.14 Scaling Challenges

- **Worker count.** Too few and you cannot use all CPUs; too many and memory blows up. Typical heuristic: one worker per CPU core, but tune empirically.
- **Connection limits.** Each WebSocket holds a connection; each SSE stream holds a connection. Load balancer and OS file descriptor limits matter at scale.
- **GC pressure.** Long-lived async workers accumulate objects. Memory-profile under load and consider periodic worker restarts.

## 5.15 Security Concerns

- Pydantic validation prevents many injection-style attacks at the request boundary.
- Streaming endpoints can leak more than intended if you forget to filter intermediate model output before sending to the client.
- Never log raw prompts or completions without redaction; PII slips in unexpectedly.
- Tool calls launched from within an LLM endpoint must enforce the *user's* permissions, not the server's. This is the most common security mistake in agent code.

## 5.16 Deployment Guide

A standard packaging path: containerize the FastAPI app, run it under uvicorn or gunicorn with uvicorn workers, expose behind a reverse proxy with TLS termination, and ship logs and metrics via OpenTelemetry. Health endpoints expose readiness and liveness. The model itself either calls out to a provider API or sits in a separate inference container (vLLM, TGI) co-located via Kubernetes. Chapter 15 details deployment in depth.

## 5.17 Monitoring Strategy

Track per-route latency, error rates, tokens in/out per request, time-to-first-token, time-to-last-token, request cancellation rate, cache hit rate, and worker memory and CPU. The two metrics most teams forget: input token p99 (which catches prompt-bloat regressions) and time-to-first-token p99 (which catches inference-server queueing).

## 5.18 Cost Optimization

Caching first. Smaller models second. Routing third — send easy requests to a small cheap model and hard ones to a frontier model. The right router can cut bills by 70% without quality loss; the wrong router cuts quality by 70% without cost savings, so measure.

## 5.19 Interview Questions

- Explain the GIL and how it affects FastAPI.
- When would you use multiprocessing instead of asyncio?
- Walk through the lifecycle of a streaming chat request.
- How does Pydantic enforce LLM output schemas?
- What changes about your architecture when you add WebSocket support?

## 5.20 Hands-on Exercises

1. Sketch on paper the components of a chat API that supports streaming, retries, and per-user rate limits.
2. Identify three places in a GenAI app where blocking code might sneak into an async handler.
3. Estimate the additional cost (in connections and memory) of upgrading 10,000 SSE clients to WebSocket clients with bidirectional traffic.

## 5.21 Common Mistakes

- Mixing sync and async without understanding which is which — calling a sync ORM in an async handler blocks the event loop.
- Forgetting cancellation. The user closed the tab; the LLM call is still running and billing.
- Treating Pydantic as a documentation tool rather than a validation contract. The whole point is failing fast on bad inputs.
- Logging unbounded fields (prompts, completions) without truncation; log storage costs explode.

## 5.22 Enterprise Best Practices

Use Pydantic for both request schemas and LLM output schemas. Standardize on FastAPI plus uvicorn workers. Treat streaming as the default response type, not an optional feature. Build a thin abstraction layer over LLM providers so swapping is a config change. Make cancellation a first-class concern in every async path that hits external services.
