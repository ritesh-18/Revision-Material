# Chapter 35 — Multi-Region and DR Architecture

## 35.1 Concept Explanation

Multi-region architecture distributes a system across multiple cloud regions to achieve goals that a single region cannot: lower latency for global users, regional regulatory compliance, and resilience against regional outages.

It is also one of the most challenging architectural undertakings. Networks have latency. Data has gravity. Consistency models break in interesting ways across thousands of miles. Operational complexity multiplies.

This chapter focuses on the SRE perspective: when to go multi-region, the architectural patterns, the failure modes, and the operational discipline required.

## 35.2 Why Multi-Region

Four reasons:

**Latency.** Users in Sydney prefer servers near Sydney. Cross-continent latency is 150-300ms; intra-region is sub-millisecond.

**Disaster recovery.** A region outage takes you down. Multi-region protects.

**Regulatory.** Data must reside in specific jurisdictions (EU resident data in EU).

**Capacity.** A single region's capacity may be limited (especially for top GPUs).

Each reason justifies different patterns. Match the architecture to the actual need.

## 35.3 Patterns

**Single region (with multi-AZ).** Most apps. Survives AZ failure. Cheap. Region outage = downtime.

**Multi-region active-passive.** Primary serves; secondary is warm standby. RPO seconds; RTO minutes. Cost: 1.5-2x.

**Multi-region active-active.** Both regions serve. Tricky for stateful systems. RPO near-zero; RTO near-zero. Cost: 2x+.

**Per-tenant pinning.** Each customer pinned to one region. Simpler than active-active. Good for regulatory.

**Cell-based.** Multiple isolated cells per region. Outage of one cell does not affect others. Used at AWS, Slack scale.

## 35.4 The Latency Reality

Cross-region latency cannot be reduced — physics.

| From | To | Latency |
|---|---|---|
| us-east to us-west | | 60-80ms |
| us-east to eu-west | | 70-90ms |
| us-east to ap-southeast | | 150-200ms |
| Within region | | <1ms |
| Same AZ | | <0.5ms |

Designs that ignore this look fast in dev (everything local) and slow in prod (multi-region calls).

The rule: cross-region calls should be rare or async. User requests should not wait on a cross-region database.

## 35.5 Data Replication

The hardest part of multi-region.

**Strong consistency cross-region** is possible (Spanner, CockroachDB) but expensive (each write needs consensus across regions; high latency).

**Async replication** is the common pattern. Writes go to primary; replicate to secondary asynchronously. RPO = replication lag.

**Multi-master / active-active write** is hardest. Conflict resolution needed. CRDTs or last-writer-wins.

**Read replicas** for read scaling across regions are common.

**Eventually consistent** stores (DynamoDB Global Tables, Cassandra) accept divergence and converge.

## 35.6 Routing

How traffic finds the right region:

**Geo-DNS** — Route 53, Cloudflare. Resolve to nearest region.

**Anycast** — same IP advertised from multiple regions. Routing picks closest. Used by Cloudflare, CDNs.

**Application routing** — service routes by user attributes.

**Manual failover** — DNS change during DR event.

Health checks integrated with routing make failover automatic.

## 35.7 Failover

What happens when primary fails:

**Automatic failover.** Health check fails; routing diverts. Fast (seconds to minutes).

**Manual failover.** Engineer triggers via runbook. Slower (5-30 minutes). Safer for complex stateful systems.

**Application-level failover.** App handles regional failures via retries and rerouting.

The pattern: automatic for stateless, manual for stateful, application-level for hybrid.

## 35.8 The Stateful Problem

Stateless services are easy multi-region. Stateful (databases, caches, queues) are hard.

For databases:
- **Multi-region primary-replica.** Single primary; replicas in other regions for reads.
- **Global databases** (Spanner, DynamoDB Global Tables, Cosmos DB) — designed for multi-region.
- **Per-region sharding** — each region owns a shard.

For caches:
- Usually per-region; warm independently.
- Cross-region cache replication is rare and expensive.

For queues:
- Per-region typically.
- Cross-region replication for events that must reach everywhere.

## 35.9 Multi-Region Failure Modes

**Split brain.** Network partition between regions; both elect leaders; conflicting writes.

**Replication lag.** Secondary far behind primary; data loss on failover.

**Asymmetric failure.** Primary up but replication broken; data accumulating without redundancy.

**Cascading failure.** Primary down; secondary takes traffic; secondary collapses under load.

Each requires specific mitigation: quorum-based consensus, lag monitoring, write-ahead validation, capacity headroom on secondaries.

## 35.10 The Cell Architecture

AWS, Slack, and others use cells:
- Each region contains many small cells.
- Each cell is fully independent (own DB, own services).
- Customers are assigned to a cell.
- Cell outage affects only that cell's customers.

Cells reduce blast radius drastically. The cost: more operational complexity and overhead per cell.

Used at companies with millions of customers where any incident's blast radius must be small.

## 35.11 Regulatory Multi-Region

Data residency drives multi-region for non-failure reasons:
- EU resident data in EU.
- Russia/China data within their borders.
- Australia government data in Australia.

Implementation:
- Per-region tenant assignment.
- Strict egress controls.
- Per-region encryption keys.
- Audit of data flows.

Often the strictest interpretation requires per-region infrastructure with no cross-region anything.

## 35.12 Cost of Multi-Region

Multi-region is expensive:
- 2x infrastructure (active-active).
- Cross-region transfer costs (significant).
- Operational overhead (more clusters, more dashboards).
- Engineering complexity.

For most products, the cost is justified only at scale or for specific compliance/availability needs.

## 35.13 Testing Multi-Region

DR drills exercise the failover path. Patterns:
- **Tabletop** — discuss the scenario.
- **Region failover drill** — actually fail to secondary.
- **Chaos** — kill primary unexpectedly.
- **Customer-impacting** — full real failover during a maintenance window.

Multi-region you have not exercised is multi-region you do not have. The first time matters.

## 35.14 Real-World Use Cases

- Stripe operates active-active across regions. Each transaction processable in any region. Notoriously well engineered.
- Many SaaS run active-passive — primary in us-east, warm in us-west. Lower cost; tolerable RTO.
- A fintech failed over to secondary during a us-east-1 outage. Worked because they had drilled it. Customers unaware.
- A team had "multi-region" config but had never tested. First real failover discovered replication had been broken for months.

## 35.15 Production Architecture

```
   Users in any region
        |
   Global DNS (geo-routing + health checks)
        |
   CDN at the edge
        |
   Region A (us-east)         Region B (eu-west)
   +-- API gateway            +-- API gateway
   +-- App tier               +-- App tier
   +-- DB primary             +-- DB replica or primary
   +-- Cache                  +-- Cache
   +-- Object store           +-- Object store (replicated)
                |                     |
              cross-region replication
                |                     |
                +---- monitoring ---- +
        |
   DR drills quarterly
        |
   Audit and documentation
```

## 35.16 Tradeoffs

| Pattern | Win | Cost |
|---|---|---|
| Single region | Cheap, simple | Region outage = down |
| Multi-region active-passive | Better DR | 1.5-2x cost |
| Multi-region active-active | Lowest RTO/RPO | High complexity, 2x+ cost |
| Cell-based | Limited blast radius | Operational overhead |
| Per-tenant pinning | Simple isolation | No cross-tenant aggregation |

## 35.17 Scaling Challenges

- Replication lag at high write volume.
- Multi-region routing complexity.
- Cross-region traffic costs.
- Operational coordination across teams.

## 35.18 Security

- Per-region encryption keys.
- Cross-region transit security.
- Audit of data flows.
- Compliance proof per region.

## 35.19 Deployment Guide

For a new multi-region rollout:
1. Define the why — DR, latency, regulatory, capacity.
2. Choose the pattern (active-passive is the safe default).
3. Identify stateful components and their replication strategy.
4. Set up routing.
5. Build observability per region.
6. Establish failover runbooks.
7. Drill quarterly.

## 35.20 Monitoring Strategy

- Per-region health.
- Replication lag.
- Cross-region request rate.
- Failover events.
- DR drill outcomes.

## 35.21 Cost Optimization

- Minimize cross-region transfer.
- Right-size secondary capacity.
- Use cheaper instance families for warm standby.
- Reserved capacity in both regions for steady-state.

## 35.22 Interview Questions

- *Compare active-passive and active-active.*
- *Walk through a region failover.*
- *Stateful data replication strategies?*
- *What is split brain and how do you defend?*
- *When is multi-region overkill?*

## 35.23 Hands-on Exercises

1. For a service, define the RPO/RTO requirements and the appropriate multi-region pattern.
2. Plan a DR drill schedule.
3. Calculate cross-region transfer cost for a service.

## 35.24 Common Mistakes

- "We have multi-region" but never tested.
- Replication lag not monitored.
- Cross-region calls in user request path.
- Active-active without consistency analysis.
- Secondary capacity insufficient for full load.

## 35.25 Enterprise Best Practices

Multi-region only where justified. Documented pattern per service. Replication lag monitored. Failover runbooks. Drills quarterly. Cross-region cost reviewed. Per-region compliance documented. Failover decision authority defined.
