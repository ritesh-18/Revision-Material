# Chapter 27 — GenAI Interviews

## 27.1 Concept Explanation

GenAI interview loops vary by company and seniority, but they cluster into a small number of categories: system design, coding, deployment and infra, MLOps, AI infrastructure, and scenario-based behavioral. This chapter prepares you for each.

A common interviewer mistake is testing pure ML theory. A common candidate mistake is showing pure ML theory without production reasoning. The intersection — production-grade GenAI engineering — is what is actually valued and what the rest of this book has prepared you for.

## 27.2 System Design Interviews

These are the most common interviews for GenAI engineers. You will be given a product ("Design a ChatGPT clone," "Design an enterprise RAG," "Design a multi-agent platform") and 45-60 minutes.

Approach:
1. **Clarify requirements** for 5 minutes. Functional, non-functional (latency, throughput, availability), constraints (budget, compliance, scale).
2. **Sketch high-level architecture.** Boxes and arrows.
3. **Dive into one or two components** the interviewer indicates.
4. **Discuss tradeoffs** at every choice.
5. **Address scaling, failure modes, cost.**

Common pitfalls:
- Jumping to specific tools before requirements.
- Skipping cost discussion.
- Single point of failure on the LLM provider.
- Ignoring evaluation and observability.
- Not addressing failure modes.

Practice with the eight designs in Chapter 22.

## 27.3 Coding Interviews

Coding interviews for GenAI engineers test fundamentals plus AI-specific patterns:

- **Async/await fluency.** Streaming endpoints, cancellation.
- **API design.** Request/response shapes, error handling, retries.
- **Data structures.** Vector similarity, ranking, deduplication.
- **Concurrency.** Worker pools, queues, backpressure.
- **Algorithms.** ANN intuition, BM25, hybrid scoring.

You will rarely be asked to implement attention from scratch. You will be asked to implement a small chat backend that streams tokens, implements rate limiting, and handles retries.

Prepare with two patterns: build a streaming API in your strongest language, and implement a simple retrieval+ranking pipeline.

## 27.4 Deployment and Infrastructure Interviews

Common questions:
- Walk through deploying an LLM service to Kubernetes.
- Handle GPU scheduling for multiple model sizes.
- Design CI/CD for a model and prompt update.
- Plan autoscaling for bursty inference traffic.
- Build observability for an AI platform.

Strong answers reference real tools (vLLM, NVIDIA GPU Operator, ArgoCD, OpenTelemetry) and explain why each is chosen.

## 27.5 MLOps Interviews

Common questions:
- How do you version a prompt?
- Walk through a fine-tuning pipeline.
- Detect drift in an LLM application.
- Design an evaluation harness.
- Compare experiment trackers.

Strong answers connect MLOps to engineering rigor (eval as CI, prompt versioning, monitoring).

## 27.6 AI Infrastructure Interviews

Targeted at platform and infra roles. Expect:
- Tensor parallelism vs pipeline parallelism.
- vLLM vs TGI vs TensorRT-LLM tradeoffs.
- KV cache management.
- GPU scheduling and MIG.
- Multi-cloud GPU sourcing.

These interviews reward depth on specific subsystems. Pick two or three areas (e.g., vLLM internals, GPU memory, Kubernetes GPU operator) and prepare to go deep.

## 27.7 Scenario-Based Questions

"You launched feature X and cost spiked 10× overnight. Walk me through your response."

"Users report the chatbot started giving wrong answers last week. How do you investigate?"

"Your inference cluster is at 95% utilization and traffic is doubling next month. What do you do?"

Strong answers describe a methodical process: detect, hypothesize, gather data, narrow, mitigate, fix, document. Reference observability, runbooks, and rollback paths.

## 27.8 Behavioral Questions

GenAI is moving fast and projects often fail. Expect:
- "Tell me about an AI project that failed and what you learned."
- "Describe a tradeoff between speed and quality you had to make."
- "How do you stay current with the field?"

Be specific, be honest about failure, and connect lessons to current practice.

## 27.9 Sample System Design Walkthrough — Enterprise RAG

A condensed example.

**Clarifications.**
- 10k internal users, 50M documents from Confluence, SharePoint.
- p95 under 3 seconds.
- SSO via Okta.
- Audit log retained 7 years.
- Data residency: EU and US.

**HLD.**
- Browser → SSO → API gateway → Chat service.
- Chat service → Retrieval service (hybrid vector + BM25 + reranker) → LLM gateway → managed model.
- Audit log to immutable store.
- Per-region deployment.

**LLD.**
- Chunks tagged with source, ACL groups, language, timestamp.
- Vector DB sharded by tenant for isolation.
- Reranker: Cohere or BGE-Reranker.
- Citations rendered as clickable references.

**Tradeoffs.**
- Hybrid retrieval over pure vector for recall.
- Managed model (Bedrock or Azure OpenAI) over self-hosted because the volume does not justify GPU ops.
- Per-region indices for residency.

**Scale.** Linear in users; document growth handled by reshard.

**Failure modes.** Provider outage → secondary provider. Vector DB partition → degraded mode (keyword only).

**Cost.** Roughly $X per query × Y queries/day = $Z/month.

This is the shape of a strong answer.

## 27.10 Sample Coding Question — Streaming Endpoint

"Sketch the request flow for a streaming chat endpoint with per-user rate limit and cancellation."

The strong answer (described in prose, since the user prefers text):

The endpoint receives a POST with the message. The handler authenticates via JWT, checks the rate limit via Redis token bucket (rejecting with 429 if exceeded), opens a server-sent event response stream, and begins streaming from the LLM client. Each token received from the LLM is forwarded to the client. The handler observes client disconnection (via the request being cancelled by the framework) and propagates cancellation to the LLM client, freeing GPU time. On completion, it logs usage to the observability pipeline and increments the user's token counter.

Trade-offs to mention: SSE vs WebSocket; backpressure handling if the client is slow; how cancellation is implemented in the chosen async runtime; what happens if the LLM call fails mid-stream.

## 27.11 Sample Behavioral Answer Pattern

"Tell me about a tradeoff between speed and quality."

"In a previous project, we shipped a feature that used a frontier model for every query. Quality was excellent but cost ran 3× our forecast. I led an analysis that showed 70% of queries were easily handled by a smaller model. We implemented a router based on input length and detected intent. Quality dropped on 5% of queries; cost dropped 60%. We built an offline replay to identify the regression cases and added them to our eval set. The lesson I took away: optimize after measuring; routing without data degrades quality. I now run a quarterly cost-vs-quality review on every AI feature I own."

Specific, honest, ends with a current practice.

## 27.12 Preparation Plan

Six weeks of focused prep for a GenAI engineer:

- **Week 1.** Read Chapters 1, 4, 8, 9, 10 of this book. Build mental models for LLM internals, embeddings, vector DBs, RAG.
- **Week 2.** Read Chapters 13, 14, 15. Understand inference systems, GPU economics, deployment.
- **Week 3.** Read Chapters 19, 20, 22. Observability, security, system design.
- **Week 4.** Practice 5 system design problems out loud with a friend.
- **Week 5.** Coding practice: streaming, async, rate limiting, retrieval.
- **Week 6.** Mock interviews, behavioral preparation, company-specific research.

## 27.13 Company-Specific Notes

- **FAANG.** Strong system design and coding; full loops including data structures.
- **AI labs (Anthropic, OpenAI, etc.).** Deep ML knowledge, research familiarity, plus production rigor.
- **Cloud providers (AWS, Azure, GCP).** Strong infra and deployment; depth on the company's own AI services.
- **AI startups.** End-to-end ownership, scrappy production stories, rapid context switching.
- **Enterprises (banks, healthcare).** Governance, compliance, integration with legacy.

Tune your stories accordingly.

## 27.14 Common Mistakes

- Memorizing definitions instead of understanding tradeoffs.
- Generic "I would use Kubernetes" without explaining why.
- Skipping evaluation and observability in designs.
- Pretending to know what you don't.
- Failing to engage with the interviewer's pushes.

## 27.15 Salary and Leveling

GenAI engineers command premium compensation in 2025-2026 due to scarcity. Expect:
- Mid (3-5 years): top-of-band software engineer compensation.
- Senior (5-10 years): senior+ software engineer, often with equity premium.
- Staff (10+ years): staff/principal level with significant equity.

Leveling differs by company. Read the job description carefully; "ML Engineer" at one company is "Software Engineer" at another.

## 27.16 Hands-on Exercises

1. Write one-page answers to five system design prompts.
2. Record yourself answering three behavioral questions; refine.
3. Build a small streaming chat backend end-to-end as a portfolio piece.

## 27.17 Common Mistakes (Career)

- Specializing too narrowly too early.
- Skipping the operational and deployment parts of the role.
- Failing to ship — interview stories are about impact.
- No public artifacts (GitHub, blog, talks) to point at.

## 27.18 Enterprise Best Practices

(Not applicable here — this chapter is candidate-side.) The mirror practice for hiring teams: standardize loops, calibrate interviewers, evaluate against a rubric, hire for trajectory not pure pattern-match. Hiring is itself a system to optimize.
