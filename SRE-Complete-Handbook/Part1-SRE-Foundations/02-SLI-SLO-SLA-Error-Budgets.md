# Chapter 2 — SLIs, SLOs, SLAs, and Error Budgets

## 2.1 Concept Explanation

This is the most important chapter in the book. Almost every SRE practice flows from the four concepts below.

**SLI (Service Level Indicator)** — a numerical measurement of some aspect of the service. "The fraction of HTTP requests that returned a 2xx response in the last 5 minutes."

**SLO (Service Level Objective)** — a target for an SLI over a window. "99.9% of requests return 2xx, measured over a rolling 30 days."

**SLA (Service Level Agreement)** — a contract with consequences. "If we deliver less than 99.5% availability in any calendar month, the customer is entitled to a 10% credit." SLAs are usually looser than SLOs (you want margin between what you promise externally and what you target internally).

**Error budget** — the inverse of the SLO. If your SLO is 99.9%, your error budget is 0.1%. This is how much *unreliability* you can spend before consequences kick in.

The genius of this framework: it turns reliability from "do your best" into a measurable resource with budgets and consequences. When the budget is healthy, you can take risks (ship features, do migrations). When it is exhausted, you must stop and improve reliability.

## 2.2 Internal Working — How SLIs Are Built

A well-designed SLI has the form: **(good events) / (valid events)** as a fraction over a window.

Examples:
- **Availability SLI** — (successful requests) / (total requests).
- **Latency SLI** — (requests served under 200ms) / (total requests).
- **Quality SLI** — (responses with all required fields) / (total responses).
- **Freshness SLI** — (queries returning data less than 5 minutes old) / (total queries).
- **Correctness SLI** — (transactions with no follow-up correction) / (total transactions).

The denominator is critical. Choose it carefully:
- Excluding requests from health checks.
- Excluding clients sending malformed input (otherwise client bugs become your SLI problem).
- Excluding traffic spikes from abusive clients.

The numerator definition is also nuanced. "Success" is what the user perceives as success, not what your code returns. A 200 OK with the wrong content is not a success.

## 2.3 Choosing the Right SLIs

Most services need only 2-5 SLIs. More than that becomes unmaintainable.

The standard menu:

- **Request-driven systems (APIs, web apps):** availability, latency, throughput, quality.
- **Pipeline systems (ETL, ML training):** freshness (how old is the output), correctness (does the output match expected), coverage (what fraction of input was processed), throughput.
- **Storage systems:** availability, latency, durability (long-term measure), throughput.
- **Stream processing:** end-to-end latency, lag (consumer behind producer), correctness.

The discipline is choosing SLIs that **users actually care about**, not metrics that are easy to measure. CPU utilization is easy to measure but no user cares about CPU. Time-to-first-byte from their perspective is what matters.

## 2.4 Setting Realistic SLOs

The two failure modes when setting SLOs:

**Too low.** "99.5% availability" sounds reasonable until users notice the 3.6 hours of downtime per month. Customer trust erodes.

**Too high.** "99.99% availability" sounds impressive until the team burns out trying to maintain a number that requires architectural rebuilds. Engineering velocity dies.

The right SLO is **achievable with current architecture and slightly tighter than current performance**. If you currently run at 99.92%, set SLO at 99.9%. If you set it at 99.99%, you'll miss every month and lose the discipline of the framework.

The standard SLO menu by service tier:

| Tier | Target Availability | Monthly Downtime Budget |
|---|---|---|
| Tier 0 (critical, revenue) | 99.99% | 4.3 minutes |
| Tier 1 (important) | 99.9% | 43 minutes |
| Tier 2 (standard) | 99.5% | 3.6 hours |
| Tier 3 (background) | 99% | 7.2 hours |
| Internal tools | 99% | 7.2 hours |

Each "nine" added roughly costs 5-10x more in engineering effort.

## 2.5 Error Budgets — The Operational Lever

If the SLO is 99.9% over a rolling 30 days, the error budget is 0.1% of requests. For a service handling 10M requests per day (300M per 30 days), the budget is 300,000 failed requests.

The budget can be **spent** in many ways:
- Bad deploys causing errors.
- Capacity shortfalls.
- Dependency outages.
- Planned migrations with risk.
- Chaos engineering experiments.

The budget is **refreshed** by good behavior — sustained reliability above SLO.

The error budget policy spells out what happens when the budget is exhausted. Typical clauses:

1. All non-critical feature work pauses.
2. The team prioritizes reliability fixes.
3. Risky changes (DB migrations, major releases) are deferred.
4. A reliability sprint is planned.
5. Only after the budget recovers does feature work resume.

Without a written policy, this never works. With one, it transforms organizational dynamics — product and engineering align around the same number.

## 2.6 Burn Rate Alerting

The naive alert: "error budget is 90% consumed." Problem: by the time you alert, you have 10% left and one bad deploy exhausts it.

The modern approach: **burn rate alerting**. Calculate how fast the budget is being consumed and alert on the rate, not the amount.

Example: if the budget is consumed at 14x the normal rate, you'll exhaust the 30-day budget in about 2 days. Page immediately.

Standard burn rate alerts:

| Severity | Burn rate | Time to exhaustion | Action |
|---|---|---|---|
| P0 page | 14.4x over 1 hour | 2 days | Page on-call now |
| P1 page | 6x over 6 hours | 5 days | Page on-call now |
| Ticket | 1x over 3 days | 30 days | Investigate during work hours |

These thresholds come from Google's experience and have become industry standard. Configure them in Prometheus with `for:` clauses to suppress flapping.

## 2.7 Multi-Window, Multi-Burn-Rate Alerts

A refinement: require the burn to be elevated in **two windows simultaneously** before paging. This avoids paging on a 2-minute blip that does not threaten the SLO.

Example: page when (5-minute burn rate > 14.4 AND 1-hour burn rate > 14.4). The short window catches fast burns; the longer window confirms it's not noise.

## 2.8 Composing SLOs for Complex Services

A user-facing flow often depends on multiple services. The end-to-end SLO is bounded by the product of dependency SLOs.

If checkout depends on auth (99.95%), inventory (99.9%), and payments (99.95%), the theoretical maximum checkout reliability is 0.9995 × 0.999 × 0.9995 = 99.8%. You cannot promise 99.95% for checkout when one of its dependencies is 99.9%.

Two strategies:
- **Negotiate stricter SLOs from dependencies** (often not possible).
- **Engineer around dependencies** (graceful degradation, fallbacks, caches) so your SLO does not strictly depend on theirs.

## 2.9 SLA Design

SLAs are external contracts. Three rules:

- **Set SLA looser than SLO.** SLO is your internal target; SLA is what you promise customers. Margin protects you.
- **Define remedies carefully.** Service credits, refunds, escalation paths. Be specific.
- **Define measurement carefully.** Whose clock? What endpoints? What excluded events (planned maintenance, force majeure)?

A common SLA structure: 99.9% availability, measured monthly, with service credits for failures, excluding planned windows announced 7 days in advance.

## 2.10 Real-World Use Cases

- A streaming service tracks "stream-start success rate" (% of play clicks that begin playing within 2 seconds) as its top SLI. Far more user-relevant than "API availability."
- A payment processor tracks "settlement freshness" — % of transactions settled within 1 hour — because users care about money landing, not about API responses.
- An ML serving platform tracks "p99 inference latency" and "model freshness" (% of requests served by current model version), because stale models cause silent quality drops.

## 2.11 Production Architecture

```
   Service emits raw events (logs, traces, metrics)
        |
   Pre-aggregation (Prometheus rules, stream processor)
        |
   SLI calculation (good events / valid events per window)
        |
   SLO comparison
        |
   Error budget computation
        |
   Burn rate calculation
        |
   Alerting (PagerDuty, Opsgenie)
        |
   Dashboards (Grafana)
        |
   Quarterly review (humans)
```

Tools that compute SLOs natively:
- **Sloth** (open-source, generates Prometheus rules from YAML SLO specs).
- **OpenSLO** (specification for SLOs as code).
- **Nobl9** (commercial SLO platform).
- **Grafana Cloud / Datadog / Honeycomb** all have SLO features built in.

## 2.12 Tradeoffs

| Choice | Win | Cost |
|---|---|---|
| Many SLIs | Comprehensive coverage | Maintenance burden, noise |
| Few SLIs | Focused | May miss user pain |
| Tight SLO | Customer happiness | Engineering effort |
| Loose SLO | Sustainable | User frustration |
| Strict error budget policy | Aligned priorities | Political friction |
| Soft error budget | Less friction | Toothless |

## 2.13 Scaling Challenges

- **Cardinality explosion** — per-customer SLOs require per-customer metrics, which can blow up Prometheus.
- **Multi-region** — global SLOs vs per-region SLOs vs per-tenant SLOs.
- **Long windows** — 30-day rolling windows require storing 30 days of high-resolution data.
- **Calibration drift** — what was "good user experience" two years ago may not be today.

## 2.14 Security Concerns

- SLO data is sensitive — it reveals when you are vulnerable.
- Customer-specific SLO data (per-tenant) raises privacy concerns.
- SLOs can be gamed if engineers know how — make the SLI definitions clear and resistant to manipulation.

## 2.15 Deployment Guide

To deploy SLOs for a service:

1. **Pick 2-3 SLIs** that map to user happiness.
2. **Instrument** at the service boundary (typically ingress or API gateway).
3. **Measure baseline** for 4-8 weeks before setting SLO.
4. **Set SLO** slightly tighter than observed baseline.
5. **Configure burn-rate alerts** with multi-window thresholds.
6. **Write error budget policy** with explicit consequences.
7. **Get leadership sign-off** on the policy.
8. **Publish dashboard** visible to all stakeholders.
9. **Review quarterly** — adjust SLO if reality has shifted.

## 2.16 Monitoring Strategy

- Real-time SLI metrics.
- Burn rate metrics (1h, 6h, 3d windows).
- Error budget remaining.
- Historical SLO compliance (month over month).
- Per-cause budget burn (deploys, dependencies, traffic spikes).

## 2.17 Cost Optimization

- Cardinality control — do not label by user ID or session ID.
- Pre-aggregate at the source — recording rules in Prometheus reduce query cost.
- Use sampling for trace-derived SLIs — full trace storage is expensive.

## 2.18 Interview Questions

- *Define SLI, SLO, SLA, error budget.*
- *Walk me through how you would set up SLOs for a new service.*
- *What is burn rate alerting and why do we prefer it?*
- *When would you decline a feature launch?*
- *How do you handle a quarter where you miss SLO?*

## 2.19 Hands-on Exercises

1. Pick a service you know. List its three most important SLIs with precise numerator and denominator definitions.
2. Calculate error budget consumed if a service serving 100M requests/month is at 99.85% over 30 days with a 99.9% SLO.
3. Design burn rate alerts for a 99.95% SLO using the standard 14.4 / 6 thresholds.
4. Draft an error budget policy for a fictional team. Include explicit consequences.

## 2.20 Common Mistakes

- Choosing SLIs based on what is easy to measure, not what users feel.
- Setting SLOs at marketing aspiration (99.99%) when reality is 99.5%.
- Including expected failures (client bugs, abuse) in the denominator.
- No multi-window burn alerts — flapping at minor blips.
- Error budget policy exists but is ignored when leadership wants a launch.
- One SLO for everything instead of per-tier.

## 2.21 Enterprise Best Practices

Every Tier 0/1 service has SLOs published and reviewed monthly. SLO compliance is part of leadership reviews. Burn rate alerts are tuned and tested via game days. Error budget policy is written, signed by VPs, and enforced. Quarterly SLO calibration ensures targets match user expectations. SLO platform is centralized (Sloth, Nobl9, Datadog) so every team uses the same definitions.
