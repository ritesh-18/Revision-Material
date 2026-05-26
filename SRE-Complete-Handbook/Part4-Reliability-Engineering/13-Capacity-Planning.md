# Chapter 13 — Capacity Planning

## 13.1 Concept Explanation

Capacity planning is the discipline of forecasting demand on a system and provisioning resources to meet it. The deliverable is an answer to: "Given expected traffic growth and engineering changes over the next quarter, what capacity do we need, and what will it cost?"

Done well, capacity planning prevents two pathological states:
- **Under-provisioning** — traffic exceeds capacity; users hit errors and slowness.
- **Over-provisioning** — capacity sits idle; money is wasted.

The right answer is neither. Capacity should be just enough to absorb planned growth plus surge headroom, refreshed as conditions change.

## 13.2 The Inputs

Capacity planning combines:

**Traffic forecasts.** What growth do we expect? From product roadmap, marketing campaigns, seasonality, customer onboarding plans.

**Performance characteristics.** How much load can each unit of infrastructure handle? From load tests, profiling, historical data.

**Reliability requirements.** What headroom for surges, failures, and growth between planning cycles?

**Cost constraints.** Budget, unit economics targets.

**Lead time.** How long does it take to provision more (minutes for cloud, weeks for on-prem, months for new GPUs)?

The product of these inputs is the capacity plan.

## 13.3 Methods for Forecasting Traffic

**Linear extrapolation.** Last quarter grew 20%; next quarter will too. Simple, works for stable products.

**Seasonal models.** E-commerce peaks at Black Friday, fitness apps peak in January. Forecast must account for cycles.

**Event-driven.** Marketing launch, partnership announcement, product release — discrete events that move traffic.

**Bottom-up.** Sum per-customer or per-tenant projections.

**Top-down.** Total addressable market × adoption rate.

Most teams use a hybrid: baseline from history, adjust for known events, validate with multiple methods.

## 13.4 Performance Characteristics

You cannot plan capacity without knowing how much load your system can take per unit.

For a stateless web service:
- Requests per second per pod (or per VM, per core).
- Latency at various load levels.
- Failure point (where latency spikes or errors begin).

For a database:
- Reads per second.
- Writes per second.
- Storage growth rate.

For a queue:
- Messages per second producible.
- Consumer throughput.
- Lag tolerance.

These numbers come from load tests (Chapter 15) and production observation.

## 13.5 Headroom

Provisioning to exact projected demand is dangerous — any surge breaks the system. Headroom is the buffer.

Typical headroom guidelines:
- **General services** — 30-50% headroom above peak forecast.
- **Latency-critical services** — 50-100% (overload latency degrades fast).
- **Slow-to-scale resources** (GPUs, large databases) — 100%+.
- **Burst-tolerant batch jobs** — 10-20% may suffice.

The right headroom depends on:
- Variance of demand (spiky vs smooth).
- Time to scale up (minutes vs weeks).
- Cost of failure (revenue loss, user trust).

## 13.6 Capacity Models

A capacity model expresses: given inputs, how much resource is needed?

Simple model:
```
   required_capacity = (peak_qps × headroom_factor) / capacity_per_unit
```

Real models account for:
- Multiple traffic types with different costs.
- Burst factors (peak / average ratio).
- Replication factors.
- Multi-region splits.
- Reserved vs on-demand mix.

The discipline is making the model explicit. Spreadsheet, Jupyter notebook, or a custom tool — the form matters less than that the model exists and can be reviewed.

## 13.7 Capacity Reviews

A typical cadence:
- **Weekly** — review utilization for any service near its limits.
- **Monthly** — review forecast vs actual; adjust capacity.
- **Quarterly** — formal review of capacity plan for the next quarter, with budget commitments.
- **Annually** — strategic capacity planning aligned with company-level forecast.

The review tracks:
- Last period's forecast vs actual (calibration).
- Current utilization.
- Next period's forecast.
- Capacity actions needed (orders, autoscale tuning, reserved instance purchases).

## 13.8 Cloud vs On-Prem Capacity Planning

**Cloud capacity** is easier — provision in minutes, pay for what you use, scale up on demand. The risk shifts from "do we have capacity" to "are we paying for unused capacity."

**On-prem capacity** is harder — months of lead time, capital expenditure, harder to rightsize. Requires more conservative planning and longer horizons.

**Hybrid** (cloud burst on top of on-prem baseline) is common at scale; combines both planning challenges.

## 13.9 Autoscaling

Autoscaling lets capacity adjust to demand automatically. Three flavors:

**Horizontal autoscaling** — add more replicas. Kubernetes HPA, AWS ASG.

**Vertical autoscaling** — give each replica more resources. Less common because it requires restarts; useful for stateful workloads.

**Cluster autoscaling** — add more nodes when pods cannot schedule. Karpenter, Cluster Autoscaler.

Autoscaling does not eliminate planning — it changes its character. You still need:
- Min replica count (so you have warm capacity).
- Max replica count (cost cap).
- Right metrics for scaling triggers.
- Reasonable scaling thresholds and cooldown periods.

A poorly tuned autoscaler can cause incidents (thrashing, cold-start storms).

## 13.10 Capacity for Spiky Workloads

Some workloads are flat (steady traffic). Others spike dramatically (flash sales, viral content, scheduled batch jobs).

For spiky workloads:
- Pre-warm before known spikes.
- Higher headroom.
- Faster-scaling infrastructure (containers over VMs, serverless for the spikiest).
- Aggressive autoscaling with sufficient max limits.
- Queue absorption for batch loads.

Spiky capacity planning is harder than flat planning by an order of magnitude.

## 13.11 GPU Capacity Planning

GPU capacity has unique constraints:
- Long lead times even in cloud (top-tier GPUs may be on waitlists).
- Cold start (model loading) is slow.
- Quantization and routing trade compute for capacity.
- Spot is often unavailable for top GPUs.

Plan further ahead. Reserve capacity. Diversify across GPU classes. Use smaller models where possible.

## 13.12 Real-World Use Cases

- A streaming service plans for Super Bowl day separately from the rest of the year. Months of preparation. 100% headroom. Pre-warmed capacity in all regions.
- A SaaS company onboards enterprise customers quarterly. Each adds known load. Capacity model uses per-customer forecasts.
- An AI company underestimated GPU demand. New customer signed and waited 6 weeks for capacity. After that, lead times became a primary capacity-planning input.

## 13.13 Production Architecture

```
   Traffic forecasts (product, marketing, sales)
        |
   Performance data (load tests, prod metrics)
        |
   Reliability requirements (SLOs, surge tolerance)
        |
   Capacity model (spreadsheet, notebook, tool)
        |
   Provisioning plan (Terraform changes, reserved purchases,
                      autoscale settings, vendor orders)
        |
   Execution
        |
   Monitoring (utilization, forecast vs actual)
        |
   Quarterly review and adjustment
```

## 13.14 Tradeoffs

| Choice | Win | Cost |
|---|---|---|
| High headroom | Surge-safe | High cost |
| Tight headroom | Cheap | Incident risk |
| Reserved instances | Lower per-hour cost | Commitment risk |
| On-demand | Flexible | Higher per-hour |
| Aggressive autoscaling | Cost-efficient | Cold-start, instability |
| Static provisioning | Predictable | Wasteful |

## 13.15 Scaling Challenges

- Forecasts get less accurate as growth accelerates.
- Multi-region capacity adds combinatorial complexity.
- Dependencies have their own capacity limits.
- Capacity changes require coordination across teams.

## 13.16 Security

- Capacity data is sensitive (reveals business plans).
- Reservation contracts need procurement involvement.
- Multi-tenant capacity must isolate noisy tenants.

## 13.17 Deployment Guide

To stand up capacity planning:
1. Inventory current capacity per service.
2. Establish baseline utilization metrics.
3. Build a simple model (spreadsheet) for the top 5 services.
4. Get forecasts from product / marketing.
5. Set headroom targets per tier.
6. Run a first quarterly review.
7. Iterate.

## 13.18 Monitoring Strategy

- Utilization per service (target a band, e.g., 40-70%).
- Headroom remaining.
- Forecast vs actual (calibration metric).
- Autoscale events (scaling up/down, cooldowns).
- Cost per unit of work.

## 13.19 Cost Optimization

- Right-size based on actual utilization.
- Use reserved instances for steady-state.
- Spot for tolerant workloads.
- Autoscale to demand.
- Quarterly cost-vs-capacity audits.

## 13.20 Interview Questions

- *How do you plan capacity for a service?*
- *Explain headroom and how to set it.*
- *When does autoscaling replace planning?*
- *Walk through a capacity model you built.*
- *Compare cloud and on-prem capacity planning.*

## 13.21 Hands-on Exercises

1. Pick a service. Build a capacity model spreadsheet: traffic forecast × per-unit cost × headroom.
2. Calculate forecast vs actual error for last quarter for one service. Adjust headroom accordingly.
3. Identify a spiky workload. Propose three improvements to handle the spikes.

## 13.22 Common Mistakes

- No model (just guess and over-provision).
- Forecast never compared to actual (no calibration).
- Autoscale tuned wrong (thrashing or never scaling).
- Single number for headroom (should differ by tier).
- Forgetting dependency capacity.

## 13.23 Enterprise Best Practices

Documented capacity model per service. Quarterly review with VP signoff for major commitments. Capacity tracked in same tooling as SLOs and cost. Forecast accuracy as a team metric. Multi-region and multi-cloud failover capacity planned and tested.
