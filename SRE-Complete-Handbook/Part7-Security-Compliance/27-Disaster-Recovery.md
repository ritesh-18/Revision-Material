# Chapter 27 — Disaster Recovery

## 27.1 Concept Explanation

Disaster recovery (DR) is the practice of preparing for and recovering from major outages: region outages, cloud provider failures, data center fires, ransomware attacks, catastrophic human error. A disaster is anything an ordinary HA setup cannot survive.

DR is unglamorous. It requires investment without immediate return. Teams that have not exercised DR are teams that will fail their first real disaster. The discipline is to invest, document, and practice — before you need it.

## 27.2 RPO and RTO

The two numbers that define DR:

**RPO (Recovery Point Objective).** How much data loss is acceptable. RPO = 1 hour means you can lose up to 1 hour of recent data.

**RTO (Recovery Time Objective).** How long to be back up. RTO = 30 minutes means full recovery within 30 minutes of disaster.

Both are business decisions, not technical. Different services have different tolerances. Critical revenue services need tight RPO/RTO; internal tools tolerate hours.

Tighter RPO/RTO costs more. Set realistic targets.

## 27.3 The DR Tiers

A spectrum:

**Tier 1 — Backup and restore.** Daily backups, restore on disaster. RPO 24 hours, RTO hours-days. Cheapest.

**Tier 2 — Pilot light.** Core systems running in DR region, scale up on failover. RPO minutes (replicated), RTO 30-60 minutes.

**Tier 3 — Warm standby.** Full system running in DR region at reduced capacity. RPO seconds-minutes, RTO 5-15 minutes.

**Tier 4 — Active-active / hot.** Both regions serve. RPO ~0, RTO ~0. Most expensive.

Different services in the same company can be in different tiers.

## 27.4 Backup Strategy

The 3-2-1 rule:
- **3 copies** of data.
- **2 different media** (disk + tape, or disk + cloud).
- **1 off-site** copy.

In cloud, this typically means:
- Production database.
- Replicated standby.
- Snapshot backups stored cross-region.
- Quarterly archive to long-term storage.

Backup details:
- **Encrypted** at rest.
- **Tested** — restore drills quarterly.
- **Retention policy** — explicit per data type.
- **Immutable** — protection against ransomware.

A backup you have never restored is not a backup.

## 27.5 Data Replication for DR

Beyond backups, real-time or near-real-time replication:

**Synchronous replication** — writes wait for replica. Strong consistency, latency cost. Limited by distance.

**Asynchronous replication** — replica catches up later. No latency penalty; some data loss risk.

**Logical replication** — events replicated, replayed. Flexible, can transform.

**Storage-level replication** — block-level mirroring. Vendor-specific.

For DR, async cross-region replication is the most common pattern. RPO equals the replication lag (typically seconds to minutes).

## 27.6 Failover Mechanisms

How traffic moves to the DR site:

**DNS-based.** Update DNS records to point to DR. TTL-sensitive; can take minutes to propagate.

**GSLB (Global Server Load Balancer).** Routes traffic based on health checks. Faster than DNS.

**Anycast.** Same IP from multiple regions; routing finds the closest healthy one. Fastest, requires infrastructure support.

**Manual failover.** Engineer triggers via a runbook. Slowest, but safest for stateful systems.

The right choice depends on what is being failed over and how comfortable the team is with automated decisions.

## 27.7 The DR Plan Document

A DR plan must be specific. A useful template:

**Scope.** Which services this plan covers.

**Tiers.** RPO/RTO per service.

**Triggers.** What constitutes a disaster requiring failover.

**Decision authority.** Who can declare and initiate failover.

**Procedures.** Step-by-step failover instructions.

**Communication.** Who tells whom, when, how.

**Rollback.** When and how to fail back.

**Test schedule.** When DR drills happen.

**Last reviewed.** Date.

The plan must be readable by a tired engineer at 3 AM. Test it.

## 27.8 DR Drills

A DR drill exercises the plan without (necessarily) impacting production.

Types:
- **Tabletop.** Discussion-based. Cheap. Catches procedural gaps.
- **Walk-through.** Manually verify each step would work.
- **Live failover.** Actually fail over to DR. Real test.
- **Full disaster simulation.** Imagine primary is gone; rebuild from scratch.

Run tabletops quarterly. Run live failovers annually or semi-annually for Tier 1 services.

The first live failover always reveals problems. That is the point.

## 27.9 Multi-Region Architecture

For high DR tiers, multi-region is necessary.

Patterns:
- **Active-passive.** Primary in one region; warm in another.
- **Active-active.** Both serve; tricky for stateful systems.
- **Cell-based.** Multiple isolated cells per region; failover is cell-level.

Multi-region complications:
- Cross-region latency.
- Replication lag.
- Cost of replicated infrastructure.
- Consistency models.
- Cross-region traffic costs.

Multi-region is not a default; it is a decision driven by RTO/RPO requirements.

## 27.10 The Crash-Only Mindset

A system designed to crash and restart cleanly is a system that recovers from disaster easily. Avoid:
- State that only exists in process memory.
- Long initialization that loses data on restart.
- "Graceful shutdown" that must run for cleanup.

Embrace:
- Idempotent operations.
- Persistent state in durable stores.
- Fast startup.
- No "warm-up" phase that data depends on.

Crash-only design makes everything from autoscaling to DR easier.

## 27.11 The Recovery Test Pattern

For high-tier services, periodically:
1. Take a backup.
2. In an isolated environment, restore from that backup.
3. Verify the data is intact.
4. Verify the service starts.
5. Run smoke tests.

This catches bit rot, missing dependencies, and configuration drift before disaster.

## 27.12 Ransomware Preparedness

Special DR concern: encrypted backups and immutable storage that ransomware cannot encrypt.

- **Air-gapped backups** — disconnected from network.
- **Immutable buckets** — write-once, cannot be modified or deleted within retention.
- **Cross-account backups** — attacker compromising production account does not have access to backup account.
- **Documented recovery procedure.**

Ransomware has become a top operational threat. Plan accordingly.

## 27.13 Cloud Region Failure

What you must answer: if AWS us-east-1 goes down completely, how long until your service is operational elsewhere?

For most companies: hours to days.
For mature DR: minutes.
For active-active: seconds.

The first time us-east-1 goes down again, the answer becomes known publicly.

## 27.14 Real-World Use Cases

- AWS us-east-1 outages have hit many companies hard. Those with mature DR survived. Those without became case studies.
- The OVHcloud datacenter fire in 2021 destroyed servers physically. Customers without backups lost data permanently.
- GitLab's 2017 incident saw an engineer delete the primary database. Backups had not been tested in months; many failed. Painful lesson.

## 27.15 Production Architecture (DR)

```
   Region A (primary)
        |
        +-- Application
        +-- Database (primary)
                  |
              async replication
                  |
                  v
   Region B (warm standby)
        +-- Application (scaled-down)
        +-- Database (replica, read-only)
        |
   Cross-region backups (S3 with replication)
        |
   Long-term archive (Glacier)
        |
   DNS / GSLB with health checks
        |
   Runbook + drill schedule + decision authority documented
```

## 27.16 Tradeoffs

| Tier | RPO/RTO | Cost | Complexity |
|---|---|---|---|
| Backup + restore | Hours/days | Low | Low |
| Pilot light | Minutes/30-60 min | Medium | Medium |
| Warm standby | Minutes/5-15 min | High | High |
| Active-active | ~0/~0 | Highest | Highest |

## 27.17 Scaling Challenges

- Replication lag grows with distance and write volume.
- Multi-region active-active has fundamental consistency challenges.
- DR drills cost real money in resources.
- Cross-cloud DR even harder.

## 27.18 Security

- DR sites need the same security posture as primary.
- Backup encryption.
- Access controls on backup data.
- Audit logging of DR actions.

## 27.19 Deployment Guide

A pragmatic DR program:
1. Tier services by RPO/RTO.
2. For each tier, define the DR mechanism.
3. Implement (start with Tier 1 services).
4. Document the plan.
5. Schedule first drill.
6. Iterate based on findings.
7. Annual review and update.

## 27.20 Monitoring Strategy

- Replication lag.
- Backup success rate.
- Time since last successful restore test.
- DR site health (it must be alive when needed).
- Time since last drill per service.

## 27.21 Cost Optimization

- Right-size DR — not every service needs Tier 4.
- Pilot light is the best cost/RTO tradeoff for many services.
- Spot instances for DR capacity (often unused).
- Tier backup storage (hot → S3, cold → Glacier).

## 27.22 Interview Questions

- *Define RPO and RTO.*
- *Compare DR tiers.*
- *Walk through a DR drill.*
- *How do you test backups?*
- *Design DR for a service.*

## 27.23 Hands-on Exercises

1. For each service you know, assign an RPO and RTO target. Defend.
2. Plan a DR drill for one critical service.
3. Restore one of your databases from backup into a test environment. Time it.

## 27.24 Common Mistakes

- Untested backups.
- Untested failover.
- DR plan that has not been reviewed in years.
- One person knows the DR plan.
- DR resources not actually capable of taking production traffic.
- Cross-region replication that no one has verified is working.

## 27.25 Enterprise Best Practices

DR tiers documented per service. Plans reviewed annually. Drills quarterly (tabletop), annually (live). Backup restores tested quarterly. Cross-region for Tier 1+. Immutable backups for ransomware protection. DR decisions tied to leadership authority. Customer-facing DR communications planned.
