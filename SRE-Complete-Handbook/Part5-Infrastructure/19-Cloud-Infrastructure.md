# Chapter 19 — Cloud Infrastructure

## 19.1 Concept Explanation

The cloud is where most modern SRE work happens. AWS, Azure, GCP, and a long tail of specialists provide the compute, storage, networking, and managed services that production systems run on. An SRE who is fluent in their cloud's primitives can move fast; one who is not depends on others to provision and operate.

This chapter covers cloud infrastructure from the SRE perspective. Less feature catalog, more "what breaks and how to think about it."

## 19.2 The Big Three

**AWS** is the dominant cloud. Broadest services, deepest enterprise adoption, most mature. Default choice unless you have a specific reason otherwise.

**Azure** is strong with Microsoft-heavy enterprises and dominant for hosted OpenAI.

**GCP** has the best networking, the strongest Kubernetes (GKE), and TPUs.

Plus secondary clouds (Oracle, Alibaba, IBM) and specialists (CoreWeave, Lambda Labs, RunPod for GPUs). Multi-cloud is increasingly common.

## 19.3 The Cloud Stack

Most services map across providers (different names, similar concepts):

| Concept | AWS | GCP | Azure |
|---|---|---|---|
| VM | EC2 | Compute Engine | VM |
| Container | ECS, EKS | GKE | AKS |
| Serverless | Lambda | Cloud Run, Functions | Functions |
| Object storage | S3 | Cloud Storage | Blob Storage |
| Block storage | EBS | Persistent Disk | Managed Disk |
| RDBMS | RDS, Aurora | Cloud SQL, Spanner | Azure SQL |
| Key-value | DynamoDB | Firestore, Bigtable | CosmosDB |
| Cache | ElastiCache | Memorystore | Cache for Redis |
| Queue | SQS, MSK | Pub/Sub | Service Bus |
| Load balancer | ELB | Cloud LB | Azure LB |
| CDN | CloudFront | Cloud CDN | Front Door |
| IAM | IAM | IAM | Entra ID |
| Secrets | Secrets Manager | Secret Manager | Key Vault |
| DNS | Route 53 | Cloud DNS | Azure DNS |
| Observability | CloudWatch | Cloud Monitoring | Monitor |
| VPC | VPC | VPC | VNet |

Learn one cloud deeply. The second is easier once you know the first.

## 19.3 Compute Choices

**VMs (EC2).** Most flexible. Full OS control. Use when you need specific kernel features, low-latency networking, or large memory/CPU.

**Containers (ECS, EKS, GKE, AKS).** Standard for most workloads.

**Serverless (Lambda, Cloud Run).** Best for spiky workloads, event-driven, low operational overhead. Higher per-request cost. Cold starts.

**Specialized (Batch, EMR, SageMaker).** For batch jobs, big data, ML training.

The pattern: containers for most things; VMs for special needs; serverless for spikiness; specialized for batch.

## 19.5 Storage Choices

**Object storage (S3, GCS, Blob).** Unbeatable cost and durability. Use for any "store and retrieve by key" workload. Standard for backups, logs, ML datasets, static assets.

**Block storage (EBS).** Attached to a single VM. Fast random access. Use for databases.

**File storage (EFS, Filestore, Azure Files).** Network filesystem. Multiple writers. Use sparingly — performance is uneven.

**Database services.** RDS, Aurora, Spanner, DynamoDB. Use these unless you have a specific reason to self-host.

Storage choice has cost and operational implications. Get it right upfront.

## 19.6 Networking

**VPC / VNet.** Your private network in the cloud. Subnets, route tables, security groups, NAT.

**Subnets.** Public (direct internet) and private (NAT-routed). Production usually places workloads in private subnets, load balancers in public.

**Security groups.** Stateful firewalls per resource. Default deny; allow specific traffic.

**NAT.** Outbound internet from private subnets. NAT gateways are expensive at high volume.

**VPC peering / Transit Gateway / VNet peering.** Connect VPCs.

**PrivateLink / Private Service Connect.** Access cloud services privately without going through internet.

**Direct Connect / ExpressRoute / Interconnect.** Dedicated lines from on-prem.

Networking is the layer where most cloud outages and misconfigurations live. Master it.

## 19.7 IAM Deeply

Every cloud's identity system has the same shape:
- **Principals** — users, groups, services.
- **Permissions** — what they can do.
- **Resources** — what they can act on.
- **Policies** — combinations.

Patterns:
- **Least privilege.** Default deny; grant only needed permissions.
- **Roles, not users, for services.** Use AssumeRole / Workload Identity / Managed Identity.
- **No long-lived access keys** in code. Use short-lived credentials.
- **Audit regularly.** Permissions drift; old roles linger.

IAM misconfiguration is the #1 cloud security risk. Treat it carefully.

## 19.8 Cost Structures

**On-demand** — pay per use, no commitment. Most flexible, most expensive.

**Reserved / Committed.** Lock in 1-3 year commitments. 30-60% discount. Risk: you pay even if you stop using.

**Spot / Preemptible / Low-Priority.** 50-90% discount, can be terminated anytime. For tolerant workloads.

**Savings Plans** (AWS) / **Committed Use Discounts** (GCP) / **Reserved VM Instances** (Azure) — different shapes of commitment.

A typical production cost structure: 60-70% reserved/committed, 20% on-demand, 10-20% spot. Track these monthly.

## 19.9 Cost Pitfalls

Things that surprise teams:
- **NAT gateway charges** — per-GB egress to internet through NAT. Adds up.
- **Cross-region transfer** — expensive. Keep regions self-contained.
- **Inter-AZ transfer** — costs a small fraction; small but accumulates.
- **CloudWatch / Cloud Logging** — high-volume logging blows up.
- **Idle resources** — unused EBS volumes, idle load balancers.
- **Data egress to internet** — most expensive direction.

Quarterly cost reviews catch these. Tools like Cost Explorer, Cloud Cost Management, FinOut, Vantage.

## 19.10 Multi-Region Architecture

Why go multi-region:
- Latency for global users.
- Disaster recovery.
- Regulatory residency.

Patterns:
- **Active-passive.** Primary region serves; secondary is warm standby.
- **Active-active.** Both regions serve. Hardest to operate correctly (multi-region writes are tricky).
- **Per-region tenant.** Each customer pinned to one region.

Multi-region adds latency between regions (cannot fix; physics). Plan accordingly.

## 19.11 Disaster Recovery

DR tiers (RPO = data loss tolerance, RTO = time to recover):
- **Backup and restore.** RPO hours, RTO hours-days. Cheapest.
- **Pilot light.** Core systems running in DR region, scale up if needed. RPO minutes, RTO 30-60 minutes.
- **Warm standby.** Full system in DR region, scaled down. RPO minutes, RTO 5-15 minutes.
- **Active-active.** RPO seconds, RTO ~0. Most expensive.

Choose by business need. Untested DR is worth nothing — exercise it.

## 19.12 The Shared Responsibility Model

Cloud provider's responsibility ends at certain layers:
- They run the data center, hardware, hypervisor.
- They patch the underlying infrastructure.
- They provide isolated tenancy.

Your responsibility:
- Configuration of services.
- Data classification and protection.
- Identity and access control.
- Application-level security.

Many cloud breaches are misconfigurations on the customer side. Read the model for your services.

## 19.13 Cloud Outages

Cloud providers have outages. AWS, Azure, GCP all do. Some major ones (memorized by SREs):
- AWS US-East-1 outages (recurring; this region is overloaded).
- Azure Active Directory outages (huge blast radius).
- GCP load balancer outages.

Plan for them:
- Multi-region for critical workloads.
- Multi-cloud for the most critical (rare; expensive).
- Stay subscribed to status pages.
- Have an incident playbook for "cloud is down."

## 19.14 Real-World Use Cases

- A startup uses AWS us-east-1 only. One region outage costs them a day of revenue. Lesson: at least multi-AZ from the start.
- A fintech runs active-active across AWS regions. When us-east-1 had a major event, they served from us-west-2 with degraded latency but no downtime.
- A government workload is on AWS GovCloud. Stricter compliance, fewer services, but mandatory.

## 19.15 Production Architecture (Cloud)

```
   Users
     |
   Cloudflare or AWS CloudFront (CDN + WAF)
     |
   Route 53 (DNS with health checks)
     |
   ALB / NLB (per region)
     |
   EKS clusters (per region)
        |
        +-- Application pods
        +-- Sidecar / mesh
        |
   RDS Aurora (multi-AZ, with cross-region read replica)
        |
   ElastiCache Redis
        |
   S3 (cross-region replication)
        |
   Secrets Manager
        |
   CloudWatch + Prometheus
        |
   IAM (Workload Identity for pods)
```

## 19.16 Tradeoffs

| Choice | Win | Cost |
|---|---|---|
| Single region | Simple | DR risk |
| Multi-region | Resilient | Complexity, cost |
| Managed services | Less ops | Lock-in, cost |
| Self-managed | Control | Ops burden |
| Reserved | Cheap | Lock-in |
| On-demand | Flexible | Premium |
| Multi-cloud | Resilience | Operational nightmare |

## 19.17 Scaling Challenges

- Single-cloud capacity limits (e.g., specific GPU instance types).
- Region-specific quotas.
- API rate limits in the cloud's control plane.
- Cross-region replication lag.
- Vendor support response time at scale.

## 19.18 Security

- IAM least privilege.
- VPC isolation.
- Encryption at rest (KMS) and in transit (TLS).
- Audit logs (CloudTrail, Cloud Audit Logs, Azure Activity Log).
- WAF and Shield / equivalent.
- DDoS protection (cloud providers offer this).
- Secrets management (never in code).

## 19.19 Deployment Guide

A first cloud architecture for a new product:
1. Choose region(s).
2. VPC with public + private subnets, multi-AZ.
3. NAT gateway for outbound (or NAT instance for cheaper).
4. Managed Kubernetes (EKS/GKE/AKS).
5. RDS or Cloud SQL.
6. ElastiCache or Memorystore.
7. S3 / GCS / Blob for object storage.
8. Route 53 / Cloud DNS / Azure DNS for DNS.
9. CloudFront / Cloud CDN / Front Door for CDN.
10. Secrets Manager / Secret Manager / Key Vault.
11. Centralized logging and monitoring.
12. Terraform for everything (Chapter 29).

## 19.20 Monitoring Strategy

- Per-service cloud metrics (CloudWatch, Cloud Monitoring).
- Quotas (often hit before you notice).
- Cost trend daily.
- IAM access patterns.
- Network throughput and errors.

## 19.21 Cost Optimization

- Right-size monthly.
- Reserved/committed for steady-state.
- Spot for tolerant workloads.
- Cross-region transfer minimized.
- Idle resource cleanup quarterly.
- Logging volume reviewed.
- Tagging discipline for cost allocation.

## 19.22 Interview Questions

- *Compare AWS, GCP, Azure for a new product.*
- *Walk through a multi-region active-passive setup.*
- *Explain the shared responsibility model.*
- *How do you optimize cloud cost without sacrificing reliability?*
- *Describe DR tiers and how to choose.*

## 19.23 Hands-on Exercises

1. Sketch the cloud architecture for a chat product at 100k users.
2. Plan DR for one critical service: RPO, RTO, mechanism, test plan.
3. Audit your IAM policies. List three overly permissive ones.

## 19.24 Common Mistakes

- Single region, single AZ for "Tier 0" service.
- Default VPC with public subnets exposing databases.
- Long-lived access keys in code or env vars.
- No cost monitoring; budget overrun discovered at month-end.
- Untested DR; first failover happens during a real incident.

## 19.25 Enterprise Best Practices

Pick one primary cloud; standardize. Terraform everything. Tag every resource (owner, env, project). Cost dashboards per team. Quarterly cost reviews. IAM access reviews quarterly. Multi-AZ by default. Multi-region for Tier 0. DR exercised twice per year. Cloud security baseline via Cloud Security Posture Management tool (Wiz, Prisma Cloud, Lacework).
