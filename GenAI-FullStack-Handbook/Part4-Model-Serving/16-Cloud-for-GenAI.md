# Chapter 16 — Cloud for GenAI

## 16.1 Concept Explanation

Cloud platforms host the vast majority of GenAI infrastructure. The choice of cloud determines GPU availability, managed AI services, networking, identity, and cost. This chapter walks the major providers, the GPU-specialist secondaries, and the GenAI-specific services each offers.

There is no universally best cloud. There are tradeoffs in price, capacity, ecosystem maturity, and where your organization already lives. Picking is a strategic decision shaped as much by hiring and procurement as by technical fit.

## 16.2 AWS for GenAI

AWS is the most widely adopted cloud for GenAI workloads. Key services:

- **EC2 GPU instances.** P5 (H100), P4d (A100), G5 (A10G), G6 (L4/L40S), Inf2/Trn1 (Inferentia/Trainium).
- **SageMaker.** End-to-end ML platform: training, inference endpoints, JumpStart for pretrained models, Pipelines for orchestration.
- **Bedrock.** Managed access to multiple foundation models (Anthropic, Meta, Mistral, Cohere, Amazon Titan) with one API.
- **Lambda.** Serverless functions, useful for thin LLM proxies, glue logic.
- **OpenSearch.** Managed search with vector capabilities.
- **EKS.** Managed Kubernetes for self-hosted inference.
- **S3.** Object storage for model weights, training data, RAG corpora.
- **CloudFront, API Gateway.** Edge and API.

Strengths: largest service catalog, deepest enterprise integrations, broadest region coverage. Weaknesses: GPU capacity is tight, pricing can be opaque, egress fees punish multi-cloud architectures.

## 16.3 Azure for GenAI

Azure has the tightest integration with OpenAI (Microsoft's exclusive cloud partner for OpenAI's hosted models). Key services:

- **Azure OpenAI Service.** Hosted OpenAI models with Azure's compliance posture, regional deployments, and private networking.
- **Azure ML.** End-to-end ML platform.
- **AI Foundry / AI Studio.** Higher-level model and agent development environment.
- **Azure Kubernetes Service (AKS).** Managed Kubernetes.
- **Cognitive Search.** Search service with vector capabilities.
- **NDv5 / ND H100 v5 / ND H200 v5.** GPU VM families.
- **Functions.** Serverless.

Strengths: OpenAI integration, strong enterprise sales motion, Microsoft 365 and Active Directory integration. Weaknesses: smaller GPU footprint than AWS in some regions, capacity constraints, complex pricing.

## 16.4 Google Cloud (GCP) for GenAI

GCP's distinctive offering is TPUs alongside GPUs. Key services:

- **Vertex AI.** Unified ML platform with model garden, training, online and batch inference.
- **Gemini API.** Google's frontier models, hosted on Vertex.
- **TPU v5e / v5p.** Tensor Processing Units for training and inference.
- **GPU VMs.** A3 / G2 / A2 families.
- **GKE.** Managed Kubernetes, widely considered the most polished of the big three.
- **Vector Search.** Managed vector database.
- **BigQuery.** Analytical database with embedded vector search and ML integration.

Strengths: best Kubernetes, TPU alternative to GPU monoculture, strong networking, integrated data + AI stack. Weaknesses: smaller GPU pool than AWS, less enterprise sales heritage.

## 16.5 Oracle Cloud (OCI)

Oracle has emerged as an aggressive GenAI player. Key advantages:

- **GPU pricing** often lower than hyperscalers.
- **Free egress** in many cases — significant for AI workloads with heavy data movement.
- **OCI Generative AI Service.** Managed model access.
- **Strong RDMA networking** for training clusters.
- **Reserved GPU capacity** that is sometimes more available than AWS/Azure.

Many AI labs and startups run training on OCI for cost reasons. Production inference often stays on AWS/Azure/GCP for ecosystem fit.

## 16.6 GPU-Specialist Clouds

A new class of GPU-specialist providers has emerged. They are not hyperscalers — fewer services, fewer regions — but compete aggressively on GPU price and availability.

- **RunPod** — on-demand and spot GPUs, simple ergonomics, popular with indie developers.
- **Lambda Labs** — AI-focused, simple billing, often-available GPUs.
- **CoreWeave** — large-scale GPU specialist with enterprise contracts.
- **Crusoe** — GPU clouds powered by stranded energy, competitive pricing.
- **FluidStack** — aggregated GPU capacity from multiple sources.
- **Together AI, Fireworks, Replicate** — model-serving platforms that abstract the cloud entirely.
- **Modal** — serverless GPU platform with fast cold starts.

For self-hosted inference, secondary GPU clouds can save 30-60% versus hyperscalers. Cost: less enterprise tooling, capacity volatility, fewer integrated services.

## 16.7 Provider Comparison

| Provider | Top GPUs | Managed AI | Strength | Weakness |
|---|---|---|---|---|
| AWS | H100, A100, Trainium | Bedrock, SageMaker | Ecosystem, regions | GPU scarcity, egress fees |
| Azure | H100, H200, ND v5 | Azure OpenAI | OpenAI access, enterprise | Capacity, pricing complexity |
| GCP | H100, A100, TPU v5 | Vertex, Gemini | Kubernetes, TPU, data | Smaller GPU pool |
| OCI | H100, H200, MI300X | OCI GenAI | Price, free egress | Smaller ecosystem |
| RunPod | H100, A100, RTX | None native | Simple, cheap on-demand | Enterprise gaps |
| Lambda | H100, A100 | None native | Simple AI focus | Capacity volatility |
| CoreWeave | H100, H200, MI300X | None native | Scale, GPU-first | Less mainstream |

## 16.8 GPU Services and Instance Selection

A right-sizing framework:

- **Small models (1-13B).** Single A10G, L40S, or RTX 4090 (consumer/dev).
- **Medium models (30B).** Single A100 80GB or H100, or 2x A10G with tensor parallelism.
- **Large models (70B).** H100 single GPU at INT4, or 2-4 GPUs at FP16, or 8x A100.
- **Frontier models (400B+).** Multi-node H100/H200 clusters with NVLink and InfiniBand.

Match GPU to traffic profile, not just to model size. A small model serving 10k QPS may need many A10G replicas; a large model serving 100 QPS may need one H100.

## 16.9 Managed AI Services

Why use managed services like Bedrock, Vertex, or Azure OpenAI?

- **No infrastructure to manage.** No GPU pools, no scaling, no incident response on inference itself.
- **Compliance posture.** SOC2, HIPAA, FedRAMP often easier through managed services.
- **Multiple models, one API.** Bedrock and Vertex offer a unified surface across providers.
- **Regional availability.** Use providers' regional deployments to meet data residency.
- **Volume discounts.** Negotiated rates at scale.

Why not?

- **Cost at scale.** Self-hosted often wins beyond a threshold.
- **Latency.** Network hops add tens to hundreds of ms.
- **Fine-tuning constraints.** Limited compared to self-hosted.
- **Vendor lock-in.** API differences are real, even with "compatible" shims.

The hybrid pattern: managed for hardest queries, self-hosted for high-volume cheap queries.

## 16.10 Networking Inside the Cloud

- **VPC (Virtual Private Cloud).** Logical network isolation.
- **Subnets.** Public (internet-facing) and private (internal).
- **NAT gateways.** Egress for private subnets. Expensive at high volume.
- **VPC Peering, Transit Gateway, Cloud Interconnect.** Connect VPCs and on-prem.
- **PrivateLink, Private Service Connect, Azure Private Link.** Access cloud services privately.
- **Direct Connect / ExpressRoute / Cloud Interconnect.** Dedicated lines from on-prem to cloud.

For AI specifically, PrivateLink-to-managed-LLM-services keeps traffic off the public internet — required for many compliance regimes.

## 16.11 IAM (Identity and Access Management)

Each cloud has its own IAM model with similar concepts:

- **Users / Service Accounts.** Identities.
- **Roles.** Sets of permissions.
- **Policies.** Permission documents attached to identities.
- **Workload Identity / IRSA.** Bind Kubernetes service accounts to cloud roles, eliminating long-lived credentials in pods.

For AI: scope permissions tightly. Models should access only what they need. Use temporary credentials. Audit logging on every model and tool action.

## 16.12 Storage

- **Object storage** (S3, GCS, Azure Blob, OCI Object Storage). Model weights, training data, RAG corpora.
- **Block storage** (EBS, GCE PD, Azure Disk). Pod-attached volumes for stateful workloads.
- **File storage** (EFS, Filestore, Azure Files). Shared filesystems.
- **Specialized** (FSx for Lustre on AWS, Hyperdisk on GCP) — high-throughput shared storage for training data.

For inference, object storage holds model weights; a fast local NVMe cache speeds loading on cold start.

## 16.13 Security

Cloud security baseline for AI:
- VPC isolation and security groups.
- Encryption at rest (cloud-managed or customer-managed keys).
- Encryption in transit (TLS everywhere).
- Secrets in cloud Secret Manager, not env vars or images.
- Audit logging (CloudTrail, Cloud Audit Logs, Azure Monitor) on every API call.
- WAF on edge traffic.
- Network policies and zero trust internal communication.
- Tenant isolation for multi-tenant AI products.

Chapter 20 covers GenAI-specific security in depth.

## 16.14 Observability

Cloud-native observability includes:
- **AWS CloudWatch, GCP Cloud Logging/Monitoring, Azure Monitor.** First-party.
- **Datadog, New Relic, Honeycomb, Grafana Cloud.** Third-party SaaS.
- **Self-hosted Prometheus/Grafana/Loki/Tempo.** Open-source stack.

For AI, the OpenTelemetry standard lets you instrument once and emit to any backend. Chapter 19 details.

## 16.15 Cost Optimization

The biggest cloud-cost levers for AI:

- **Reserved instances / committed use** for steady state (1-year typically saves 30-50%, 3-year more).
- **Spot / preemptible** for batch and stateless workloads (50-90% savings).
- **Right-sizing** — many teams over-provision GPUs.
- **Egress** — keep traffic in-region; use private connections for cross-region.
- **Storage tiering** — move cold data to cheaper tiers.
- **Multi-cloud arbitrage** — run batch workloads where GPU price is lowest.
- **Idle GPU detection** — track utilization; shut down idle replicas.

Budget alarms per service. Review the top 10 cost drivers monthly. Set anomaly detection on cost — sudden 5x spike usually means a leak.

## 16.16 Multi-Region Strategy

Reasons to deploy multi-region: latency for global users, regulatory residency, disaster recovery, GPU capacity arbitrage.

Patterns:
- **Per-region inference clusters** with regional model deployments.
- **Geo-DNS routing** to the nearest region.
- **Failover regions** with smaller standby capacity.
- **Per-region data isolation** for residency compliance.

Watch cross-region egress costs and replication lag for RAG indices.

## 16.17 Hybrid and Multi-Cloud Patterns

- **Burst to cloud.** On-prem baseline plus cloud burst capacity.
- **Cloud primary, on-prem disaster recovery.** Reverse of burst.
- **Multi-cloud active-active.** Geo-routed traffic across multiple clouds.
- **Cloud-of-clouds.** Aggregator services (model serving platforms) abstract clouds entirely.

Most teams start single-cloud and expand only when motivated by clear capacity, cost, or regulatory pressure.

## 16.18 Real-World Architecture

```
   User
      |
   DNS (Cloudflare or Route 53 with health checks)
      |
   CDN (Cloudflare or CloudFront)
      |
   API gateway (cloud-managed or Kong)
      |
   App tier (EKS / GKE / AKS)
      |
   +--- Self-hosted inference cluster (vLLM on H100s)
   +--- Managed inference fallback (Bedrock / Vertex / Azure OpenAI)
   +--- Vector DB (managed Pinecone or self-hosted Qdrant on cluster)
   +--- Postgres (RDS / Cloud SQL / Azure SQL)
   +--- Redis (ElastiCache / Memorystore)
   +--- Object storage (S3 / GCS / Blob)
   +--- Observability (CloudWatch + Grafana + OTEL)
   +--- Secrets (Secrets Manager + Vault)
   +--- CI/CD (GitHub Actions + ArgoCD)
   +--- Infra as Code (Terraform)
```

## 16.19 Tradeoffs

| Choice | Win | Cost |
|---|---|---|
| Hyperscaler | Service breadth, compliance | GPU scarcity, premium |
| Specialist GPU cloud | Cheap GPUs | Fewer services |
| Managed model API | Zero ops | Higher per-token cost |
| Self-hosted | Cheaper at scale | Ops burden |
| Multi-cloud | Resilience | Complexity, egress |

## 16.20 Interview Questions

- Compare Bedrock, Vertex, and Azure OpenAI.
- When would you choose a specialist GPU cloud over a hyperscaler?
- Design a multi-region failover for a global chat product.
- Walk through cost optimization for a $1M/year GPU bill.
- Where does PrivateLink fit in a compliance-bound architecture?

## 16.21 Hands-on Exercises

1. Price a 70B inference deployment across three clouds for 10M tokens/day.
2. Plan a regional rollout map for a global product, accounting for GPU availability.
3. Design the IAM model for an agent platform with 20 tool integrations.

## 16.22 Common Mistakes

- Forgetting egress fees in multi-cloud designs.
- Pinning to one provider's proprietary APIs (lock-in).
- Over-provisioning GPUs.
- Skipping reserved capacity for steady state.
- Letting credentials live in env vars when workload identity is available.

## 16.23 Enterprise Best Practices

Standardize on one primary cloud; use specialists tactically. Terraform everything; manual cloud changes drift fast. Tag every resource with owner, project, environment. Centralize log and metric aggregation across clouds. Negotiate annual commitments tied to projected usage. Review cost monthly and capacity quarterly.
