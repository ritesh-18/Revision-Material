# Chapter 29 — Startup and Enterprise Architecture

## 29.1 Concept Explanation

Architecture choices for a 10-person startup differ from a 10,000-person enterprise. The same engineer in different contexts should make different decisions. This chapter contrasts the two ends of the spectrum and the path between them.

The principle: match architecture to your true constraints. A startup's true constraint is time-to-market; an enterprise's is risk and integration. Optimizing the wrong constraint is the most common architecture mistake at every scale.

## 29.2 The Startup MVP Architecture

Stage: 1-5 engineers. Goal: ship something users will actually use. Budget: tight.

```
   Browser (Next.js on Vercel)
        |
   Backend (FastAPI or Next.js API routes)
        |
   Managed services
     +-- OpenAI / Anthropic / Bedrock
     +-- Pinecone or pgvector on managed Postgres
     +-- Supabase or Neon for relational
     +-- Upstash or managed Redis
     +-- Sentry for errors, PostHog for analytics
        |
   Cloudflare for DNS, CDN, WAF
```

Principles:
- Managed everything. Your time is more expensive than the cloud bill.
- Single region. Latency tradeoffs can wait.
- One model provider. Failover can wait.
- No Kubernetes. Vercel, Render, Fly.io, Railway are enough.
- Simple observability. Sentry + provider dashboards.
- Ship in weeks, not months.

What you skip without guilt: Kubernetes, service mesh, multi-cloud, fine-tuning, self-hosted inference, complex MLOps.

## 29.3 Startup at 10k Users

Stage: 5-15 engineers. Goal: reliability and unit economics.

Changes from MVP:
- Add a real LLM gateway (Portkey, LiteLLM, or homegrown) for routing and observability.
- Add semantic and prefix caching.
- Move from notebooks to a proper eval harness.
- Add real observability (LangSmith or Langfuse + Grafana).
- Begin model routing — cheap model for easy queries.
- Start cost dashboards.
- First retrospective on architecture.

What you still skip: self-hosted inference (still cheaper to buy than build at this scale), full enterprise compliance posture, multi-region.

## 29.4 Startup at 100k Users

Stage: 15-50 engineers. Goal: scale to a million.

Changes:
- Kubernetes (managed) becomes worth it.
- Self-host the highest-volume models (often a quantized small open model).
- Multi-region deployments for global users.
- SOC 2 compliance work begins.
- Dedicated platform team forms.
- Multi-provider for resilience.
- Cost optimization becomes a discipline.

## 29.5 Startup at 1M Users

Stage: 50+ engineers. Goal: efficiency, reliability, expansion.

Changes:
- Significant self-hosted inference fleet.
- Multi-cloud experiments.
- Dedicated AI platform internal product.
- Fine-tuning becomes worth it for specific tasks.
- Compliance broadens (HIPAA, FedRAMP, regional).
- Engineering becomes specialized (infra, MLOps, application, frontend, etc.).

## 29.6 The Enterprise Architecture

Stage: thousands of engineers across many teams. Goal: enable AI across the company safely.

Pattern: build a centralized AI platform; teams build on it.

```
   Teams (dozens) build product features
        |
   Internal AI Platform (a product itself)
        |
   +-- LLM Gateway (routing, caching, observability)
   +-- Embedding Service
   +-- Vector DB (multi-tenant)
   +-- Prompt Registry
   +-- Eval Harness
   +-- Governance Policy Engine
   +-- Cost Accounting
        |
   Underlying Infrastructure
   +-- Managed K8s clusters (per region)
   +-- GPU pools (mixed sizes, reserved + spot)
   +-- Multi-cloud where strategic
   +-- Observability mesh
   +-- Security and compliance tooling
```

The platform team's goal: every team can ship an AI feature without reinventing the wheel.

## 29.7 The Path From Startup to Enterprise

Recognizable inflection points:

- **First non-engineer user.** Documentation must improve.
- **First multi-tenant customer.** Isolation becomes serious.
- **First compliance audit.** Logging and governance solidify.
- **First on-call rotation.** Operational maturity required.
- **First platform team.** Federated AI patterns emerge.
- **First international launch.** Multi-region and data residency.
- **First fine-tune in production.** MLOps grows up.

Each inflection invalidates some MVP-era choices. Plan for them, don't fight them.

## 29.8 FAANG-Level AI Architecture

At hyperscaler scale, AI infrastructure is its own product line. Distinctive patterns:

- Custom hardware accelerators (TPUs, Inferentia, Trainium).
- Internal model training at frontier scale.
- Massive RAG corpora with custom retrieval infrastructure.
- Bespoke serving frameworks (often building on or alongside open-source).
- Dedicated platform teams numbering in hundreds.
- Multi-region deployments with sub-second failover.
- Compliance posture for every major regulatory regime.
- Internal evaluation infrastructure at scale.

Most engineers will not work at this scale. The patterns at this scale inform standards everyone else inherits.

## 29.9 The Cost-Efficient Startup AI Stack

For a startup wanting maximum capability on minimum budget:

- Cloudflare for DNS, CDN, WAF — generous free tier.
- Vercel or Cloudflare Pages for frontend.
- Supabase or Neon for relational data.
- Upstash for Redis and queues.
- pgvector or Pinecone serverless for vectors.
- OpenAI/Anthropic via direct API, with model routing.
- Sentry for errors.
- PostHog for product analytics.
- GitHub Actions for CI.

Total fixed cost under $200/month. Variable cost dominated by LLM tokens. Scales to first thousand users without changes.

## 29.10 The Mid-Stage AI Stack

For a company at hundreds of thousands of users:

- AWS or GCP as primary cloud.
- Managed Kubernetes (EKS or GKE).
- Qdrant or Pinecone for vectors.
- Managed Postgres and Redis.
- LiteLLM or Portkey as LLM gateway.
- LangSmith or Langfuse for AI observability.
- Grafana + Prometheus + Loki for infra observability.
- ArgoCD for GitOps.
- Self-hosted vLLM for the hottest model.
- OpenAI/Anthropic for the rest.

Monthly cost in the tens of thousands. Reliability matters; ops discipline emerges.

## 29.11 The Enterprise AI Stack

At enterprise scale:

- Multi-region Kubernetes with service mesh.
- Hybrid managed + self-hosted models.
- Multi-cloud with primary and secondary.
- Internal AI platform team.
- Vault for secrets.
- SIEM for audit logs.
- Full OpenTelemetry coverage.
- Comprehensive eval and red-team harnesses.
- Compliance automation.

Monthly cost in the millions. Headcount in dozens or hundreds focused on platform.

## 29.12 Team Structure

Startup: full-stack engineers everywhere.

Growth: specialists emerge — frontend, backend, infra, ML.

Mid-stage: platform team forms. Product teams consume platform.

Enterprise: many platform teams (infra, MLOps, AI services, security, observability). Central architecture function. Multiple AI CoEs.

## 29.13 Operational Maturity

Startup: founder is on-call. Incidents are messy and personal.

Growth: rotation forms; postmortems start.

Mid-stage: SLOs documented. Runbooks built. Fire drills quarterly.

Enterprise: 24/7 follow-the-sun on-call. SREs dedicated. Reliability is a measured discipline.

## 29.14 Tradeoffs

| Approach | Win | Cost |
|---|---|---|
| Managed everything (startup) | Velocity | Future ceiling |
| Self-hosted everything | Control | Engineering time |
| Centralized platform | Consistency | Coordination overhead |
| Federated teams | Velocity | Inconsistency |
| Multi-cloud | Resilience | Complexity |
| Single cloud | Simplicity | Lock-in |

## 29.15 Common Mistakes

- Premature Kubernetes adoption.
- Over-engineering for scale you do not have.
- Skipping observability "until later."
- Building a platform before customer demand justifies it.
- Treating enterprise patterns as universal.

## 29.16 Real-World Use Cases

- A two-person startup ships a working AI assistant in six weeks on Vercel and OpenAI; reaches 10k users with no infra hires.
- A 200-person company decides to self-host their highest-volume model after measuring cost; cuts bill by 60% and adds two infra engineers to operate it.
- An enterprise builds a centralized platform; sees AI feature velocity across the company double within a year.

## 29.17 Interview Questions

- Walk me through what changes architecturally when going from 1k to 1M users.
- When do you self-host?
- How would you organize an AI engineering team of 50?
- Compare centralized and federated AI architectures.
- What is the right MVP stack for a GenAI startup?

## 29.18 Hands-on Exercises

1. Pick a hypothetical product. Sketch the MVP architecture in detail.
2. Now sketch the 100k-user version. Mark which choices changed.
3. Sketch the enterprise version. Mark the new components.

## 29.19 Enterprise Best Practices

Match architecture to stage. Plan for next inflection, not current one. Document decisions so future engineers understand why. Invest in the platform team only when product teams' duplicated work justifies it. Treat reliability and cost as engineering disciplines, not afterthoughts.
