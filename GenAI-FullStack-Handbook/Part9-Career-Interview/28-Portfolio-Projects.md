# Chapter 28 — Portfolio Projects

## 28.1 Concept Explanation

Portfolio projects are how you prove you can do the work. A polished resume gets you a screening call; a portfolio gets you a job offer. For GenAI engineers, the bar has risen — interviewers expect to see real deployments, real observability, real cost analysis, not just a notebook with a chat demo.

This chapter walks through six portfolio projects, each progressively more ambitious. For each, the architecture, deployment, observability, scaling, CI/CD, and infra are sketched. Build at least one to depth; build all six over time.

## 28.2 Why Portfolio Projects Matter

- They demonstrate end-to-end thinking that interviews cannot fully test.
- They prove you have shipped, not just studied.
- They give you stories — "in my agent project, I encountered X and solved it with Y."
- They establish reputation. Public projects with users compound over time.

The depth-vs-breadth choice: one polished project is better than four half-finished ones. Pick one and ship it well.

## 28.3 Project 1 — AI Chat App

**Goal.** Production-quality chat application backed by an open or closed LLM, deployed to a cloud, instrumented with observability.

**Architecture.**
- Next.js frontend with streaming chat UI.
- FastAPI backend with SSE streaming, JWT auth, rate limiting.
- Redis for sessions and rate limits.
- Postgres for conversation history.
- LLM gateway calling OpenAI/Anthropic with provider failover.

**Deployment.** Containerized; deployed to a managed cloud (Vercel + Render, or full Kubernetes if you want extra credit).

**Observability.** OpenTelemetry across frontend and backend. Grafana dashboard with TTFT, error rate, token usage, cost.

**Scaling.** Demonstrate handling 100 concurrent users. Add autoscaling rules and document the choice.

**CI/CD.** GitHub Actions: build, test, deploy on main.

**Infra.** Terraform for cloud resources.

**What to demonstrate.** Streaming, cancellation, retries, observability, cost dashboard. Not just "it works" but "I know why each piece is there."

## 28.4 Project 2 — Enterprise RAG

**Goal.** Production-quality RAG over a meaningful document corpus (your own notes, an open dataset, a public docs site).

**Architecture.**
- Document loaders for the source.
- Chunker (structure-aware).
- Embedding via an open model (BGE) or API.
- Qdrant or pgvector for storage.
- Hybrid retrieval with BM25.
- Cross-encoder reranker.
- LLM generation with citations.
- Eval harness with at least 50 questions and answers.

**Deployment.** Same as Project 1 plus the RAG ingestion pipeline.

**Observability.** Track retrieval hit rate, reranker scores, citation accuracy, LLM-as-judge eval scores.

**Scaling.** Demonstrate handling 100k documents. Document index build time, query latency, and cost.

**CI/CD.** Add eval gates — PRs that regress eval scores below a threshold fail.

**Infra.** Vector DB as a managed service or a self-hosted pod with persistent volume.

**What to demonstrate.** Hybrid retrieval, reranking, citations, evaluations, tenant isolation if multi-user. The full RAG production stack.

## 28.5 Project 3 — Multi-Agent System

**Goal.** A system that orchestrates multiple LLM agents to accomplish a multi-step task.

**Architecture.**
- Planner agent that decomposes a goal.
- Specialist agents (research, code, write, review).
- Tool layer with MCP integrations.
- Workflow engine (Temporal or Inngest) for durability.
- Shared memory in Redis + vector DB.

**Pick a real task.** "Research and write a comparative article on X." "Refactor a Python module given a description." "Plan and book a multi-stop trip given constraints." A specific use case beats a generic framework.

**Deployment.** Kubernetes with workflow workers and agent services.

**Observability.** Trace every agent step. Track cost per goal. Monitor termination patterns.

**Scaling.** Demonstrate handling 10 concurrent goals end-to-end.

**CI/CD.** Eval harness for end-to-end goal completion rate.

**Infra.** Temporal cluster (use Temporal Cloud or self-host).

**What to demonstrate.** Real agent reliability practices: step limits, termination detection, tool permissions, durable retries, observability of multi-step flows.

## 28.6 Project 4 — Voice AI Assistant

**Goal.** Real-time voice interaction with an AI assistant. Push-to-talk minimum; full duplex interruption-aware is the stretch goal.

**Architecture.**
- WebRTC or WebSocket audio capture in the browser.
- Streaming STT via Deepgram or Whisper streaming.
- LLM with streaming responses.
- Streaming TTS via ElevenLabs, Cartesia, or OpenAI.
- Voice activity detection and interruption handling.

**Deployment.** Streaming-heavy infrastructure; latency budgets dominate.

**Observability.** End-to-end latency from user speech to assistant audio start. STT WER, LLM latency, TTS latency.

**Scaling.** Demonstrate ten concurrent calls.

**CI/CD.** Automated tests on STT/TTS quality with recorded samples.

**Infra.** Media server (LiveKit is a common choice) or WebSocket-based audio handling.

**What to demonstrate.** Sub-second responsiveness, natural interruption, multilingual support, graceful degradation when one of the three pipelines fails.

## 28.7 Project 5 — AI Code Reviewer

**Goal.** An AI bot that reviews pull requests and posts comments with suggestions.

**Architecture.**
- GitHub or GitLab webhook receives PR events.
- Backend fetches diff and surrounding context.
- LLM (or specialist code model) generates review comments.
- Comments posted via API with line references.
- Optional: cached previous reviews for similar diffs.

**Deployment.** Webhook handler service plus background worker.

**Observability.** Comments per PR, acceptance rate (devs marking comments resolved), latency from PR open to first comment.

**Scaling.** Handle bursty PR traffic.

**CI/CD.** Eval suite of PRs with known issues; measure detection rate.

**Infra.** Lightweight stateless service.

**What to demonstrate.** Practical AI integration with developer workflows; quality measured by human signals; cost discipline (avoid commenting on every line).

## 28.8 Project 6 — AI Workflow Engine

**Goal.** A no-code platform where users design workflows combining AI steps, tool steps, conditional branches, and human approval.

**Architecture.**
- Workflow definition UI (React).
- Backend that compiles definitions to a workflow engine.
- Workflow engine (Temporal) executes.
- Built-in tool catalog: LLM call, HTTP call, vector search, human approval.
- Audit log per run.

**Deployment.** Full stack with persistent storage.

**Observability.** Per-run trace, per-step latency and cost.

**Scaling.** Handle 100 concurrent workflows.

**CI/CD.** Standard.

**Infra.** Temporal cluster, Postgres, object store.

**What to demonstrate.** Combining AI with classical software in a maintainable way; designing for non-engineers; thinking about workflow safety and idempotency.

## 28.9 Common Requirements Across Projects

Each portfolio project should include:
- **README** with architecture diagram, deployment instructions, and demo video.
- **Live demo** if possible.
- **Observability dashboard** screenshots showing real metrics.
- **Eval results** with methodology.
- **Cost analysis.**
- **What I learned** section.
- **What I would do next** section.

## 28.10 Build Approach

A schedule that works:

- **Week 1.** Scope, requirements, architecture sketch. Set up the repo, CI, basic skeleton.
- **Week 2-3.** Core functionality. Make the happy path work end to end.
- **Week 4.** Observability, eval, error handling, edge cases.
- **Week 5.** Polish, write the README, record a demo.
- **Week 6.** Publish. Write a blog post. Share for feedback.

Six weeks per project; one project per quarter if working part-time on top of a job.

## 28.11 Publishing and Promotion

A portfolio project unseen is half a portfolio project. Publish on:
- GitHub (well-documented).
- A personal blog.
- LinkedIn (post the demo video).
- Hacker News (Show HN) for the bigger ones.
- Engineering communities (subreddits, Discord, Twitter, conferences).

Engagement is bonus content for interviews. "This repo got 500 stars" and "users gave this feedback" are credible signals.

## 28.12 Avoiding Portfolio Anti-Patterns

- **Pure notebook demos** with no deployment.
- **No observability** or eval — "it works, trust me."
- **Copies of existing tutorials** without distinctive contribution.
- **Over-engineering** for a portfolio scale (you do not need Istio).
- **Half-finished projects** abandoned mid-way.

## 28.13 Tradeoffs

| Choice | Win | Cost |
|---|---|---|
| Deep on one project | Memorable, demonstrates rigor | Less breadth |
| Many small projects | More keywords | Each is shallow |
| Bleeding-edge tech | Distinctive | Possibly unreliable |
| Boring choices | Reliable, employable | Less differentiating |

## 28.14 Interview Use of Portfolio

Bring the portfolio into interviews. "When discussing system design, I encountered this exact problem in my X project — here's how I solved it." Specific, credible, prepared.

## 28.15 Common Mistakes

- Building yet another generic chatbot. Differentiate.
- Skipping deployment.
- Skipping observability.
- No documentation; reviewers can't tell what you built.
- Hidden in a private repo.

## 28.16 Enterprise Best Practices

(For hiring managers reviewing portfolios.) Look for: end-to-end ownership, evidence of measurement, willingness to address failure modes, documentation quality. A polished portfolio is a strong signal of how the candidate will work on the job.
