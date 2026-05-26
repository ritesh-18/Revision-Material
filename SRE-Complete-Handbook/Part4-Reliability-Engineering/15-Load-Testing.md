# Chapter 15 — Load Testing

## 15.1 Concept Explanation

Load testing is the practice of generating synthetic load against a system to measure its behavior under realistic or stressed conditions. It is how you turn "I think we can handle the launch" into "we have data showing we can handle 5x the launch."

Three reasons to load test:
1. **Capacity validation** — confirm planned capacity meets demand.
2. **Bottleneck discovery** — find weakest link before users do.
3. **Regression prevention** — catch performance regressions in CI before deploy.

A system that has never been load-tested has unknown behavior under load. Hope is not a strategy.

## 15.2 Types of Load Tests

**Smoke test.** Light load to verify the system functions. Run on every deploy.

**Load test.** Expected production load to verify SLOs hold. Run before launches.

**Stress test.** Beyond expected load to find breaking points. Run periodically.

**Soak test.** Sustained load over hours/days to find memory leaks, resource exhaustion.

**Spike test.** Sudden load increase to test autoscaling and burst handling.

**Breakpoint test.** Steadily increasing load to find the failure threshold.

Each type answers a different question. Most production-grade systems should run several.

## 15.3 What to Measure

For each load test, capture:
- Throughput (requests/sec achieved).
- Latency (p50, p95, p99, p99.9).
- Error rate by type.
- Resource utilization (CPU, memory, network, disk).
- Database / cache metrics.
- Dependency latency.

Run with full observability so you can connect symptoms (slow requests) to causes (DB saturation).

## 15.4 Load Generation Tools

**k6** (Grafana) — modern, JavaScript-based, excellent reporting. Becoming the default.

**Locust** — Python-based, scriptable, good for complex scenarios.

**JMeter** — old, mature, complex GUI but powerful.

**Gatling** — Scala-based, high performance.

**wrk / wrk2** — minimal CLI, very fast for HTTP. Great for quick tests.

**Artillery** — Node.js based, scriptable.

**Vegeta** — Go, simple, fast.

For most teams, k6 is the modern choice — programmable, integrates with CI, exports to Prometheus/Grafana.

## 15.5 Test Scenarios

A realistic load test scenario:
- **Mix of operations** — not just one endpoint; realistic distribution.
- **Realistic user behavior** — sessions with multiple actions, think times.
- **Variable inputs** — different users, different products, different sizes.
- **Realistic geographies** — load from real client regions.

A test that hammers one endpoint at constant rate is easy but unrepresentative. Production traffic is messier; tests should reflect that.

## 15.6 Test Environments

Where to run load tests:

**Production** — most realistic, highest risk. Use feature flags, off-peak hours, and conservative limits.

**Staging** — production-like, lower risk. Must be sized like production for results to translate.

**Dedicated load-test environment** — clean, expensive, used for big tests.

**Ephemeral environments** — spin up for the test, tear down after. Cloud-native pattern.

Best practice: smoke tests in CI on small ephemeral envs; full load tests in staging or production-with-flags.

## 15.7 Load Test in CI

A modern practice: include performance tests in CI. Each PR runs a small load test against a small ephemeral environment. Regressions caught before merge.

This works best for:
- Stateless services with predictable performance.
- Critical paths where regressions are expensive.
- Teams with disciplined test data management.

Less practical for systems with complex dependencies or seasonal data.

## 15.8 Production Load Testing

Testing in production is the gold standard for realism. Done carelessly, it causes outages. Done carefully, it builds confidence.

Practices:
- **Shadow traffic** — duplicate production traffic to a test instance, do not serve responses to users.
- **Canary at higher load** — send a tiny percentage of users at higher-than-normal rates.
- **Tenant isolation** — load test on a synthetic tenant.
- **Off-peak windows** — early morning, weekends.
- **Kill switch** — ability to stop the test instantly.

Companies known for production load testing: Netflix (chaos engineering as a sibling discipline), Stripe (gradual launch ramps), Google (large-scale traffic experiments).

## 15.9 Result Analysis

A load test produces graphs. Reading them:

**Latency over time.** Spikes mean something. Correlate with deploys, GC pauses, autoscaling events.

**Throughput vs offered load.** Should rise linearly until saturation, then flatten or decline.

**Error rate vs load.** Should be ~0 until saturation, then climb.

**Resource utilization vs load.** Reveals which resource is the bottleneck.

Compare to baselines. A test in isolation is data; a test compared to prior tests is signal.

## 15.10 The Coordinated Omission Problem

A classic load testing pitfall. Naive load tests have the generator wait for each response before sending the next. When the system slows, the test slows with it — *masking the latency it was supposed to measure*.

Modern tools (k6, wrk2, Gatling) use **closed-loop** generation with **open-loop** semantics — the test continues sending requests at the target rate even when responses are slow. This produces realistic latency measurements.

If you are using a tool that does not handle coordinated omission, you are not measuring what you think you are.

## 15.11 Real-World Use Cases

- An e-commerce platform load tests for Black Friday in October. Reveals database connection limits would be hit. Increases pool size, validates fix.
- A startup launches a Product Hunt feature. Has not load tested. Site melts at 5x normal traffic. Embarrassing. Lesson: load test before launches.
- A team's CI runs a 5-minute load test on every PR. Catches a regression where someone removed a cache, before merge.

## 15.12 Production Architecture

```
   Test scenarios (k6 / Locust scripts)
        |
   Load generators (cloud VMs / k6 cloud / dedicated cluster)
        |
   System under test
        |
   Observability (Prometheus / Tempo / Loki / Grafana)
        |
   Test report (k6 metrics + Grafana dashboards)
        |
   Comparison to baseline (regression detection)
        |
   Verdict: pass / fail / investigate
```

## 15.13 Tradeoffs

| Choice | Win | Cost |
|---|---|---|
| CI load tests | Fast feedback | CI compute cost |
| Staging tests | Safe | Drift from prod |
| Production tests | Realistic | Risk of impact |
| Long tests | Catch leaks | Time |
| Short tests | Fast | Miss slow failures |
| Synthetic load | Reproducible | May not reflect real users |
| Replayed real traffic | Realistic | Privacy / complexity |

## 15.14 Scaling Challenges

- Generating very high load requires distributed generators.
- Test data management for stateful tests.
- Reproducibility — yesterday's test does not always repeat.
- Coordinating with downstream service owners.

## 15.15 Security

- Load tests can mimic DoS — coordinate with infra teams.
- Test data must not include real PII.
- Production tests need legal / compliance signoff.

## 15.16 Deployment Guide

A practical adoption path:
1. Start with smoke tests in CI (small load, basic validation).
2. Add weekly staging load tests.
3. Build a baseline of expected performance.
4. Add regression gates to CI.
5. Plan production load tests for major launches.

## 15.17 Monitoring Strategy

- Load test results compared to baseline.
- Trend over time of key metrics.
- Failures during load tests (incident-like investigation).
- Resource costs of testing.

## 15.18 Cost Optimization

- Use ephemeral environments (only pay during tests).
- k6 cloud or self-hosted depending on volume.
- Schedule tests in off-peak times.
- Reuse test data and scripts.

## 15.19 Interview Questions

- *Walk through a load test you ran. What did you learn?*
- *Compare load, stress, soak, and spike tests.*
- *What is coordinated omission?*
- *How do you load test in production safely?*
- *What metrics do you capture during a load test?*

## 15.20 Hands-on Exercises

1. Write a k6 script for a small API. Run it locally; read the report.
2. Set up a baseline + regression detection for a CI load test.
3. Plan a production load test for a hypothetical Black Friday launch.

## 15.21 Common Mistakes

- Single-endpoint constant-rate tests (unrealistic).
- Missing coordinated omission handling.
- Testing only in staging, then surprised by prod behavior.
- No baseline comparison (test results don't mean much standalone).
- Skipping observability during tests.
- Tests no one reads.

## 15.22 Enterprise Best Practices

CI smoke tests on every PR. Staging load tests weekly. Production load tests before major launches. Baselines tracked and visualized. Coordinated omission handled by tool choice. Test plans reviewed in production readiness reviews. Load test outcomes feed capacity planning.
