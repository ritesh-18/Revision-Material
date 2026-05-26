# Chapter 33 — Cost Optimization (FinOps)

## 33.1 Concept Explanation

FinOps is the practice of bringing financial accountability to cloud spend. It is the discipline of treating cloud cost as an engineering metric — observable, attributable, optimizable — rather than an opaque bill that arrives monthly.

For SREs, FinOps is the natural extension of capacity planning. You know what the system needs; FinOps asks how to provide it cheaply. The skill is balancing cost against reliability and performance.

The cloud's flexibility is also its risk. Auto-scaling, on-demand instances, and pay-per-use pricing make it easy to spend much more than necessary. A 30-50% cost reduction is common when a team applies FinOps for the first time.

## 33.2 The FinOps Framework

The FinOps Foundation defines three phases:

**Inform.** Visibility into what is being spent and why.

**Optimize.** Reduce cost without sacrificing function.

**Operate.** Embed cost in everyday engineering decisions.

Most teams stop at Inform (dashboards). Mature teams cycle through all three continuously.

## 33.3 Cost Visibility

You cannot optimize what you cannot see. Foundations:

**Tagging.** Every resource tagged with owner, project, environment, cost center. Cloud APIs aggregate by tags.

**Cost allocation.** Map tags to teams and budgets. Cross-charge or showback.

**Dashboards.** Per-team, per-service, per-environment cost trends. Daily granularity.

**Anomaly detection.** Alert on sudden spikes (10x in a day usually means a leak).

**Budgets.** Explicit per-team budgets with alerts at 50%, 80%, 100%.

Tools: AWS Cost Explorer, GCP Billing, Azure Cost Management, third-party (CloudHealth, Vantage, Cloudability, FinOut, Apptio).

## 33.4 The Big Cost Drivers

For most cloud workloads:
1. **Compute** (EC2, GKE, Lambda) — usually 40-60% of bill.
2. **Storage** (EBS, S3, RDS storage) — 10-20%.
3. **Network egress** — 5-15%, can be much higher.
4. **Managed databases / services** — 10-20%.
5. **Monitoring / logging** — 5-15%, surprisingly large.
6. **Other** — load balancers, KMS, support, etc.

Spend optimization effort proportional to spend. The top three drivers usually deserve most attention.

## 33.5 Compute Optimization

**Right-sizing.** Most fleets run at 20-40% utilization. Identify and shrink over-provisioned resources.

**Spot / preemptible.** 50-90% discount. For tolerant workloads (batch, stateless web).

**Reserved / committed.** 30-60% discount for steady-state.

**Savings Plans / CUDs.** Flexible commitments.

**Newer instance families.** Often cheaper per unit; e.g., AWS Graviton (ARM) is 20-30% cheaper than Intel.

**Auto-scaling.** Scale down off-peak.

**Bin-packing.** More pods per node via better scheduling.

## 33.6 Storage Optimization

**Tier by access pattern.**
- Hot (S3 Standard, EBS gp3): frequent access.
- Warm (S3 IA, gp3 lower IOPS): occasional.
- Cold (S3 Glacier): rare.
- Archive (Glacier Deep): never (almost).

**Lifecycle policies.** Auto-tier based on age.

**Cleanup.** Unattached EBS volumes, old snapshots, abandoned S3 buckets.

**Compression** for stored data.

**Deduplication** where applicable.

**Right-size database storage** — provisioned IOPS often over-set.

## 33.7 Network Cost

Easy to miss; can be enormous.

**Egress to internet** — expensive. Use CDN.

**Cross-region** — expensive. Keep regions self-contained.

**Cross-AZ** — small but adds up at scale.

**NAT gateway** — per-GB charge. NAT instances or VPC endpoints can be cheaper.

**Private connectivity** (Direct Connect, ExpressRoute) — can reduce egress for high-volume.

A common surprise: log shipping cross-region. Multi-region apps' log centralization is often the egress culprit.

## 33.8 Observability Cost

Observability spend grows faster than infrastructure.

**Metrics.** Cardinality is the dominant driver. Drop unused metrics. Aggregate.

**Logs.** Volume is the driver. Sample. Tier hot/warm/cold. Use Loki over Elasticsearch where possible.

**Traces.** Sample aggressively. Most traces are uninteresting.

**Managed observability** (Datadog, New Relic) can rival compute spend. Audit usage; consolidate.

## 33.9 Right-sizing Process

A pragmatic monthly cadence:
1. Pull utilization data for last 30 days.
2. Identify resources at <40% utilization.
3. For each, propose a smaller size.
4. Validate via load test or simulation.
5. Apply.

Tools: AWS Compute Optimizer, GCP Recommender, Azure Advisor, kubectl-resource-recommender.

Automation can do this continuously (VPA for K8s; Karpenter for nodes).

## 33.10 Reservations Strategy

For steady-state workloads:
- **Baseline** what you definitely need (lowest of last 12 months).
- **Reserve** that capacity for 1-3 years.
- **Variable** above baseline goes on-demand or spot.

Standard pattern: 50-70% reserved/committed for baseline, 20-30% on-demand, 10-20% spot.

The pitfall: reserve more than you need. Track reservation utilization monthly.

## 33.11 Cost Aware Engineering

Cost is an engineering concern, not just finance. Patterns:

**Cost in design reviews.** New systems include estimated cost.

**Per-feature cost.** Track cost per feature; expensive features get scrutiny.

**Cost-of-failure.** Outages have direct cost (compute, lost revenue, time).

**Build vs buy.** Often build looks cheaper because cloud cost is invisible; FinOps surfaces this.

Engineers who think about cost write more efficient systems.

## 33.12 The FinOps Team

At sufficient scale, a dedicated FinOps team emerges. Responsibilities:
- Visibility tooling.
- Reservation purchasing.
- Vendor negotiation.
- Cross-team coordination.
- Anomaly response.

The team is small (1-5 people for most companies). High leverage when done right.

## 33.13 Vendor Negotiation

Cloud bills above $1M/year deserve negotiation:
- Enterprise Discount Programs (EDPs).
- Private pricing agreements.
- Annual commit discounts.
- Service-specific commitments.
- Multi-year deals with locked rates.

These conversations are real. Major customers get 20-40% off list rates.

## 33.14 AI Workload Cost

AI workloads have unique cost characteristics:

**GPUs are expensive.** $4-30/hour. Idle GPUs are pure cost.

**Token-based pricing** for APIs. Watch input vs output cost.

**Model size vs traffic.** Self-hosted breaks even with API at certain volume.

**Quantization** halves cost with minimal quality loss.

**Caching** is the cheapest LLM cost cut.

Chapter 25 of the GenAI handbook covers this in depth.

## 33.15 Real-World Use Cases

- A startup spent $200k/month on AWS. After 3 months of FinOps work: $90k. Right-sizing, reserved instances, log volume cleanup.
- A team's NAT gateway cost $30k/month from accidental cross-region traffic. Fixed routing; cost dropped to $2k.
- An ML team ran always-on GPUs at 10% utilization. Migrated to scale-to-zero; saved 80% of GPU cost.

## 33.16 Production Architecture (FinOps)

```
   Cloud billing data
        |
   Cost data warehouse (BigQuery, Snowflake)
        |
   Tagging-based attribution
        |
   Per-team dashboards
        |
   Anomaly detection
        |
   Budgets with alerts
        |
   Monthly review meetings
        |
   Action items tracked
        |
   Reservation strategy reviewed quarterly
        |
   Vendor commit reviewed annually
```

## 33.17 Tradeoffs

| Choice | Win | Cost |
|---|---|---|
| Aggressive optimization | Lower bill | Engineering time |
| Light optimization | More engineering | Higher bill |
| Spot instances | Cheap | Interruption |
| Reserved | Cheaper | Commitment |
| Self-hosted obs | Cheaper at scale | Ops |
| Managed obs | Less ops | High at scale |

## 33.18 Scaling Challenges

- Cost reviews don't scale linearly with team size.
- Tag discipline breaks down without enforcement.
- Vendor commits at scale require finance involvement.
- Multi-cloud cost aggregation.

## 33.19 Security

- Cost data is sensitive (reveals scale, growth).
- Access controls on cost tools.
- Anomalies can indicate compromise (cryptomining).

## 33.20 Deployment Guide

Starting FinOps:
1. Get tagging baseline.
2. Set up cost dashboards.
3. Configure budgets and alerts.
4. Right-size top spend items.
5. Buy reservations for steady-state.
6. Monthly review meetings.
7. Form FinOps function at scale.

## 33.21 Monitoring Strategy

- Daily cost trend per service.
- Per-tag attribution.
- Budget burn rates.
- Reservation utilization.
- Anomaly alerts.
- Cost per business metric (cost per user, cost per order).

## 33.22 Cost of FinOps

Tools cost: $0 (cloud-native) to $50k+/year (enterprise platforms).
People cost: $0 (part-time SRE) to a dedicated team.

Almost always net-positive if done seriously.

## 33.23 Interview Questions

- *Walk through a FinOps program you'd implement.*
- *How do you reduce cloud cost without harming reliability?*
- *Compare reserved instances and spot.*
- *What's your approach to observability cost?*
- *How do you attribute cost in a multi-tenant cluster?*

## 33.24 Hands-on Exercises

1. For a cloud account, identify the top 10 resources by cost.
2. Estimate savings from right-sizing the top 3.
3. Calculate the breakeven for self-hosting one currently managed service.

## 33.25 Common Mistakes

- No tagging.
- Reservations purchased without analysis.
- Cost discussed only at month-end.
- Engineers not aware of their service's cost.
- Treating cloud as infinite (it is, but it's not free).
- Optimizing in the wrong place (saving on storage while compute burns).

## 33.26 Enterprise Best Practices

Mandatory tagging. Cost dashboards per team. Monthly reviews. FinOps team at scale. Vendor negotiations annually. Cost in design reviews. DORA-like metrics for cost (cost per deployment, cost per user). Cost responsibility owned by engineering, not finance alone.
