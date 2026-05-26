# Chapter 22 — Failure Modes

## 22.1 Concept Explanation

This chapter catalogs the failure modes SREs encounter repeatedly. Each pattern is real; each has caused famous outages. Internalizing them lets you design systems that avoid them and recognize them quickly when they happen.

The meta-lesson: failure modes are predictable. The same kinds of failures keep happening to different companies. Reading other companies' postmortems teaches you to spot these patterns in your own systems before they break.

## 22.2 Single Point of Failure (SPOF)

A component whose failure brings down the whole system.

Classic SPOFs:
- One database instance.
- One load balancer.
- One DNS provider.
- One identity provider.
- One CI/CD system blocking all deploys.
- One person who knows the system.

Fix: redundancy and failover. But add it carefully — adding redundancy often introduces new failure modes (split brain, consistency issues).

## 22.3 Cascading Failure

One failure triggers others, amplifying impact. Common pattern:
1. A dependency slows down.
2. Callers' threads block waiting.
3. Threads accumulate.
4. Callers run out of capacity, slow down or fail.
5. Their callers experience the same.
6. The whole system grinds to a halt.

Defenses:
- Timeouts at every layer.
- Circuit breakers.
- Bulkheads (isolated resource pools).
- Load shedding.
- Backpressure.

Cascading failures are the most dangerous because they amplify minor incidents into major ones.

## 22.4 Thundering Herd

Many clients hit the same resource at the same time.

Triggers:
- Cache entry expires; all clients hit the source.
- A service comes back online after outage; queued retries hit at once.
- A scheduled cron job runs simultaneously across many machines.
- A popular event (game launch, sale) causes traffic spike.

Defenses:
- Jittered retries (random delay).
- Distributed locks for cache warming.
- Probabilistic cache refresh ahead of expiry.
- Rate limiting at the dependency.

## 22.5 Slow Failures (Brown-outs)

Not "down" but "slow." Often worse than outright failure because:
- Health checks pass.
- Retries make things slower.
- Capacity drains as threads stick.

Detection:
- Latency-based alerts, not just availability.
- p99 matters.
- Circuit breakers that trip on slow responses, not just errors.

## 22.6 Retry Storms

A failing dependency causes retries. Retries amplify load. Load makes the dependency fail more. Spiral.

Defenses:
- Bounded retry counts (3 is plenty).
- Exponential backoff.
- Jitter.
- Circuit breakers stop retries entirely.
- Retry budgets (cap total retry rate as fraction of normal traffic).

A common saying: "retries are the only thing more dangerous than no retries."

## 22.7 Configuration Bombs

A bad config change deployed quickly to all instances.

Examples:
- A regex change that consumes 100% CPU.
- A timeout set to 0.
- A new feature flag enabled globally.
- A DNS change pointing at the wrong region.

Defenses:
- Gradual rollout of config (canary configs).
- Validation in CI.
- Feature flags with kill switches.
- "Two-person rule" for high-blast-radius changes.

## 22.8 Memory Leaks

Memory usage grows over time. Eventually OOM.

Diagnosis:
- Memory metric trending up.
- Pod restart frequency increasing.
- OOM kills in logs.

Causes:
- Unbounded caches.
- Connection or thread accumulation.
- Reference cycles in some languages.
- Heap fragmentation.

Mitigation:
- Restart policies as a stopgap.
- Memory profiling.
- Code review for common leak patterns.

## 22.9 Connection / Resource Exhaustion

Limited resources run out:
- File descriptors.
- Database connections.
- HTTP connection pools.
- TCP ports.
- Thread pool slots.

Symptoms:
- New connections fail.
- Errors about "too many open files," "connection refused," "no available threads."
- Existing requests OK, new ones queue or fail.

Fixes:
- Increase limits.
- Connection pooling.
- Properly close resources.
- Connection multiplexing (HTTP/2).

## 22.10 Split Brain

A network partition causes both sides to elect a leader. Both serve writes. When the partition heals, conflicting state.

Defenses:
- Quorum-based consensus (majority required).
- STONITH ("shoot the other node in the head") in older HA systems.
- Fencing tokens.
- Avoid multi-master where possible.

## 22.11 Clock Skew

Servers have different times. Code that compares timestamps across servers breaks.

Symptoms:
- Tokens issued in the future or past.
- Events ordered incorrectly.
- Authentication failures (token "not yet valid").

Defenses:
- NTP / chrony for sync.
- Tolerance windows in time-sensitive logic.
- Logical clocks where appropriate.
- TrueTime (Spanner) for tight ordering.

## 22.12 Daylight Saving / Timezone Bugs

A surprisingly common failure: code that assumes time monotonically increases or that timezone conversions are simple.

Mitigations:
- Store all times in UTC.
- Convert to local only at presentation.
- Test around DST transitions.
- Use libraries for time zone handling.

## 22.13 Race Conditions

Multiple processes operate on shared state without coordination, producing inconsistent results.

Symptoms:
- Intermittent bugs that only happen under load.
- Data inconsistencies you cannot explain.
- "Cannot reproduce" issues.

Defenses:
- Use database transactions.
- Use locks (carefully — deadlock risk).
- Optimistic concurrency control.
- Idempotent operations.
- Single-writer designs.

## 22.14 Deadlock

Two or more processes wait on each other forever.

Symptoms:
- System hangs.
- Specific operations time out.
- No CPU activity but no progress.

Detection:
- Thread dumps.
- Lock monitoring.
- Timeouts.

Prevention:
- Acquire locks in consistent order.
- Use timeouts on locks.
- Detect cycles in lock graphs.

## 22.15 Resource Contention

Two workloads compete for the same resource. Both slow down.

Examples:
- Two services on the same node compete for CPU.
- Multiple processes hitting the same disk.
- Background and foreground workloads on the same database.

Defenses:
- Resource limits (cgroups, request/limit in K8s).
- Workload isolation (separate clusters or namespaces).
- Priority classes (preempt low-priority work).

## 22.16 Data Corruption

Data is lost or wrong. Worse than downtime in some ways.

Causes:
- Bug in code writes wrong data.
- Storage hardware failure.
- Concurrent writes without coordination.
- Bad migration.
- Rollback that does not roll back data.

Defenses:
- Checksums.
- Backups (with regular restore tests).
- Audit logs of mutations.
- Schema validation.
- Database constraints.

## 22.17 Vendor / Dependency Outage

A third-party service is down. Your service depends on it. You are partially down.

Examples:
- Payment provider outage.
- Cloud provider region.
- Auth provider.
- DNS provider.

Defenses:
- Multiple vendors where critical.
- Graceful degradation (read-only mode, cached responses).
- Async paths where sync isn't strictly needed.
- Status page subscriptions for awareness.

## 22.18 The Three Sigma Failure

A rare event that exceeds your planning. 100-year storm; viral content; CEO mentions your product on TV.

Defenses:
- Headroom (Chapter 13).
- Autoscaling (with appropriate maximums).
- Graceful degradation under extreme load.
- Pre-built "DDoS mode" or read-only mode.

You cannot plan for all surprises, but you can plan for the system to fail gracefully.

## 22.19 The "Push the Big Red Button" Test

For every major failure mode, ask: do we have a one-click mitigation?
- Bad deploy → one-click rollback.
- Bad config → kill switch.
- Bad provider → failover.
- Bad service → take it out of rotation.
- Bad query → kill it.
- Surge load → degraded mode.

If the answer is "we'd have to figure it out at 3 AM," fix that before you need it.

## 22.20 Real-World Use Cases

- Cloudflare's regex outage (2019) was a config bomb caught only after global impact.
- The GitLab database deletion incident (2017) was a tired engineer + missing backups.
- The Knight Capital incident (2012) was a deploy bug that lost $440M in 45 minutes.
- Atlassian's 2022 outage was a buggy script causing 14 days of customer-impacting data loss.

Each teaches a different lesson. All are required reading.

## 22.21 Interview Questions

- *Walk through a cascading failure.*
- *What is split brain and how do you prevent it?*
- *Describe a thundering herd.*
- *How do you defend against retry storms?*
- *Compare data corruption to outage as a failure type.*

## 22.22 Hands-on Exercises

1. For your service, list the top five failure modes likely to occur. For each, identify the mitigation.
2. Identify three single points of failure in your architecture.
3. Design a graceful degradation plan for a critical service.

## 22.23 Common Mistakes

- Designing for the happy path only.
- Unbounded retries without backoff.
- No timeouts on dependencies.
- Single regions / single AZs for critical paths.
- "We have backups" — never tested restore.
- "We can roll back" — never tested rollback.

## 22.24 Enterprise Best Practices

Failure modes catalog per system. Game days exercising each. Postmortems trend analysis to spot recurring patterns. Production readiness reviews check for SPOFs and missing mitigations. DR exercises annually. Failure mode training as part of onboarding.
