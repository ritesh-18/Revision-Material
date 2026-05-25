# Chapter 22 — AI System Design

## 22.1 Concept Explanation

System design for GenAI is the practice of taking a product idea and producing a working architecture — components, data flows, scaling story, failure modes, cost envelope. The output is a diagram you can defend in front of skeptical engineers, plus a written rationale for every choice.

This chapter walks through canonical AI system designs: ChatGPT-like products, enterprise RAG, copilots, multi-agent platforms, voice AI, AI search, AI IDEs, AI workflow systems. Each section follows the same pattern: requirements, high-level design, low-level design, tradeoffs, scaling, bottlenecks, cost.

## 22.2 Design Method

A reliable approach for any AI system design:

1. **Requirements.** Functional (what it does), non-functional (latency, throughput, availability, compliance), and constraints (budget, team size, timeline).
2. **High-level design.** Boxes and arrows. Identify components, their responsibilities, and the data flow.
3. **Low-level design.** APIs between components, data schemas, key algorithms.
4. **Tradeoffs.** Every choice has alternatives; name them.
5. **Scaling.** What changes at 10×, 100×, 1000× traffic.
6. **Bottlenecks.** What breaks first, what breaks worst, what is easy to fix.
7. **Cost.** Order-of-magnitude estimate.

You will be evaluated on the *reasoning*, not the diagram. Saying "I would use Pinecone" is weaker than "I would start with pgvector because we already operate Postgres and our scale is under 10M vectors; I would move to Pinecone or Qdrant past that scale because dedicated vector DBs handle high-cardinality filtering better."

## 22.3 Design 1 — A ChatGPT-Like System

**Requirements.** Multi-turn chat, streaming responses, conversation history, code rendering, file uploads, sub-2s time-to-first-token, 10M monthly active users.

**HLD.**
```
   Browser <-- SSE -- Edge load balancer -- API gateway --
   Chat service -- LLM gateway -- Inference pool
                |
                +-- Postgres (users, conversations)
                +-- Redis (sessions, rate limits, cache)
                +-- Object store (file uploads)
                +-- Event bus (logs, billing)
                +-- Observability
```

**LLD.**
- Conversation stored as event log; current state derived.
- Streaming via SSE; one connection per active turn.
- Rate limit by user and by org via Redis token bucket.
- Inference on a mix of model sizes; router decides.
- Files uploaded to object store with presigned URLs; embedded in background.

**Tradeoffs.** SSE over WebSocket (simpler; suffices for one-way streaming). Postgres for conversation state (consistency matters). Redis for rate limits and ephemeral session (latency matters). Self-hosted inference for hot models; provider fallback for surge.

**Scale.** Hot path is the inference pool; scale by replicas. Vector DB if RAG is added. Edge by region for global latency.

**Bottlenecks.** GPU capacity, KV cache concurrency, long-tail latency from long conversations.

**Cost.** Inference dominates. With efficient batching and routing, $0.001-0.005 per resolved turn. 10M MAU × 5 turns/day average × 30 days = 1.5B turns/month. At $0.002 average: $3M/month inference cost.

## 22.4 Design 2 — Enterprise RAG

**Requirements.** Knowledge assistant for 5000-person company. Document corpus 1M files (Confluence, SharePoint, Slack). ACL-aware. p95 under 3 seconds. SSO. Audit logs. Data residency in two regions.

**HLD.**
```
   User browser
        |
   SSO -> API gateway -> Chat service
        |
   Retrieval service (hybrid: vector + BM25 + reranker)
        |
   Vector DB (per-region, ACL-tagged)
   BM25 index
   Reranker (Cohere or self-hosted)
   LLM gateway -> regional inference
        |
   Audit log -> append-only store
        |
   Ingestion pipeline (connectors, parser, chunker, embedder)
```

**LLD.**
- Chunks tagged with source, ACL groups, language, timestamp.
- Retrieval filters by user's effective groups before vector search.
- Reranker runs over top 50 candidates; top 10 to the LLM.
- Citations rendered as clickable references.
- Audit log captures: who asked what, what was retrieved, what was returned.

**Tradeoffs.** Hybrid retrieval (better recall than pure vector). Per-region indices for residency. Cross-encoder reranker (latency cost) is worth it for quality. Cohere Rerank vs self-hosted BGE-Reranker: managed for simplicity, self-hosted for cost at scale.

**Scale.** Vector DB sharded by source or tenant. Inference scales with QPS. Ingestion is batch + streaming for new content.

**Bottlenecks.** ACL filtering performance; retrieval reranker latency; ingestion freshness for time-sensitive docs.

**Cost.** Embedding ingestion is one-time + delta. Storage moderate. Inference proportional to user activity. Reranker calls add 20-30% to per-query cost.

## 22.5 Design 3 — AI Copilot in an IDE

**Requirements.** Inline code suggestions, multi-line completions, chat panel, refactor commands, sub-500ms suggestion latency, support 50 languages.

**HLD.**
```
   IDE plugin -- HTTPS / WebSocket -- Edge -- Auth -- Copilot service
   --> Context builder (open files, project structure)
   --> LLM gateway
        +-- Small fast model for inline (FIM: fill-in-middle)
        +-- Large model for chat / refactor
   --> Result post-processor (cancel-aware)
```

**LLD.**
- Context window assembled from current file + nearby files + project header + cursor position.
- Specialized FIM (fill-in-middle) prompting for inline suggestions.
- Aggressive cancellation: user types another keystroke, cancel inflight suggestion.
- Telemetry: acceptance, edit-after-accept, rejection patterns.

**Tradeoffs.** Tiny specialized model for inline vs general LLM (inline needs sub-500ms). Streaming vs full completion (streaming gives perceived speed but inline UX wants the whole suggestion at once). Self-hosted (cost) vs API (capability).

**Scale.** QPS very high for inline (every keystroke can trigger). Burst tolerance critical. Cache common patterns.

**Bottlenecks.** Latency, latency, latency. Cold model load, network RTT, model inference time. Aggressive caching and locality.

**Cost.** Volume × per-call cost. Most copilots subsidize cost from subscriptions.

## 22.6 Design 4 — Multi-Agent Platform

**Requirements.** Users describe goals in natural language; the system orchestrates multiple specialized agents (research, code, design, communication) to deliver outcomes. Long-running (minutes to hours). Resumable. Auditable.

**HLD.**
```
   User -> Goal intake
        |
   Planner agent (decomposes goal into sub-tasks)
        |
   Workflow engine (Temporal) tracks task graph
        |
   Specialist agents (one per domain)
        |
   Tool layer (MCP servers + custom tools)
        |
   Shared memory (vector DB + structured)
        |
   Output assembler
        |
   Human-in-the-loop checkpoints
```

**LLD.**
- Each agent is a deterministic loop of (read state, call LLM, optionally call tools, write state).
- Workflow engine durably stores state; resumable on crash.
- Specialist agents have scoped tools (code agent can run sandboxed code; research agent can browse).
- Inter-agent messages structured (typed schemas).
- Costs tracked per agent run and per goal.

**Tradeoffs.** Durable workflows vs in-memory loops. Hierarchical orchestration vs peer-to-peer. Strict role separation vs flexible delegation.

**Scale.** Throughput in goals/minute. Cost per goal varies wildly with complexity. Backpressure essential.

**Bottlenecks.** Compound failure (one agent's wrong output poisons downstream). Cost runaway. Debug complexity.

**Cost.** Hard. A goal might be $0.10 or $50.00. Per-user budget enforcement is mandatory.

## 22.7 Design 5 — Voice AI Assistant

**Requirements.** Sub-300ms latency from user speech to assistant response start. Natural interruption. Multilingual. Hosts thousands of concurrent calls.

**HLD.**
```
   Phone / WebRTC -> Media server
                          |
            +-- Streaming STT (Deepgram, Whisper streaming) --+
            |                                                 |
            +-- Voice activity detection + interrupt handling +
                          |
                  LLM with streaming
                          |
            +-- Streaming TTS (ElevenLabs, Cartesia) ---------+
                          |
                  Audio back to caller
```

**LLD.**
- Audio streams in WebRTC chunks (20ms typical).
- STT produces partial transcripts; LLM is called when end-of-utterance detected.
- LLM streaming responses tokenized into sentences and sent to TTS chunk by chunk.
- VAD detects new user speech mid-response and cancels.
- Tool calls (e.g., "book me a meeting") run server-side with confirmation prompts.

**Tradeoffs.** Cascaded (STT+LLM+TTS) vs end-to-end speech model. Cascaded is more flexible and currently lower-latency in production; end-to-end is the future.

**Scale.** Concurrent calls × audio bandwidth. Voice infrastructure is heavy (encoding, transport).

**Bottlenecks.** Latency every millisecond matters. Cold starts. Tail latency on STT or TTS spoils experience.

**Cost.** Per minute of conversation. STT, LLM, and TTS each add cost. Often $0.05-0.30 per minute fully loaded.

## 22.8 Design 6 — AI Search Engine

**Requirements.** Web-scale search with conversational interface. Returns answers with citations. Sub-2s end-to-end.

**HLD.**
```
   Query -> Query understanding (intent, decomposition)
         -> Web index search (keyword + vector + freshness signals)
         -> Page fetcher (parallel)
         -> Page parser + extractor
         -> Retrieval + reranking
         -> LLM with citations
         -> Result with sourced answer
```

**LLD.**
- Web index built and maintained by crawler infrastructure (out of scope for many startups; use Bing or Brave APIs).
- Fast first pass: keyword + a small set of candidate URLs.
- Fetch top candidates concurrently with strict timeouts.
- Extract main content; chunk; embed; rerank against query.
- LLM composes answer with citation numbers.

**Tradeoffs.** Self-built crawler (massive investment) vs search API (per-query cost). Cache aggressively; many queries repeat.

**Scale.** Latency dominated by page fetch. Aggressive parallelism, strict per-page timeouts, fallback when fetches fail.

**Bottlenecks.** Long-tail slow pages, freshness vs latency, hallucination from irrelevant fetched content.

**Cost.** Search API + LLM + bandwidth. Per-query cost is the bill driver.

## 22.9 Design 7 — AI IDE / Code Editor

Beyond a copilot, an AI-first IDE understands the whole codebase, runs commands, edits files, and reviews PRs.

```
   Editor -- gRPC -- Agent service
                          |
                          +-- Repo indexer (embeddings + symbol graph)
                          +-- Code execution sandbox
                          +-- File system tools
                          +-- Test runner
                          +-- Git integration
                          +-- LLM gateway
```

Distinctive challenges:
- Whole-repo context that cannot fit in any context window — selective retrieval.
- Edit application that respects file structure (diff-aware).
- Long-running sessions with growing context.
- Strict permission control for destructive operations.

## 22.10 Design 8 — AI Workflow Engine

A platform where non-engineers build automations using AI agents and tools.

```
   Designer UI -> Workflow definition (DAG)
                       |
                  Workflow runtime (Temporal)
                       |
        +--- LLM steps ----+
        +--- Tool steps ----+
        +--- Human approval +
        +--- Conditional branches
                       |
                  Audit trail
```

Challenges: schema versioning of workflow definitions, observability per workflow run, cost tracking per workflow, security boundary per tool.

## 22.11 Cross-Cutting Concerns

Across all designs:
- **Observability** end to end.
- **Auth and rate limit** at the edge.
- **Tenant isolation** at every persistence layer.
- **Cost accounting** per request and per tenant.
- **Eval harness** for every model and prompt path.
- **Failure modes** documented per dependency.
- **Region awareness** for compliance and latency.

## 22.12 Tradeoffs

| Decision | Lean toward | When |
|---|---|---|
| Provider API | Faster start, less ops | Early stage, low volume |
| Self-hosted | Lower unit cost, more control | Mature, high volume |
| Workflow engine | Long, multi-step processes | Agents, batch jobs |
| Streaming UX | Perceived speed matters | Chat, voice, copilots |
| Strong consistency | State must be correct | Conversations, billing |
| Eventual consistency | Latency matters more | Caches, indices |

## 22.13 Cost Estimation Method

For any system, estimate by:
1. **Traffic.** Users × sessions/user × queries/session × tokens/query.
2. **Unit cost.** Per-token at chosen model, plus fixed costs (vector DB, infra).
3. **Inflation.** Add 30% for caches, retries, batch ingestion.
4. **Headroom.** Plan for 3× peak vs average.

Order of magnitude is usually enough in design. Exact numbers come from measuring.

## 22.14 Interview Tips

When asked to design an AI system:
- Start by clarifying requirements before drawing.
- Sketch the high-level boxes early so the interviewer can redirect.
- Justify every choice with a tradeoff, not "I would use X."
- Explicitly call out failure modes and how you handle them.
- Mention cost early and often.
- Discuss what changes at scale.

## 22.15 Hands-on Exercises

1. Pick one of the eight designs and write a one-page architecture doc covering all 17 elements of the chapter template.
2. Estimate cost for that design at 10k users and at 10M users.
3. Identify the three biggest risks and how you would mitigate each.

## 22.16 Common Mistakes

- Jumping to tools before requirements.
- Ignoring failure modes.
- Skipping cost estimation.
- Single point of failure on a single model provider.
- Tightly coupling prompt/model to application code.

## 22.17 Enterprise Best Practices

Design docs reviewed before building. Cost envelopes signed off by finance early. Tenant isolation explicitly designed, not assumed. Failure modes documented and exercised. Model and prompt swappability built in. Observability planned upfront.
