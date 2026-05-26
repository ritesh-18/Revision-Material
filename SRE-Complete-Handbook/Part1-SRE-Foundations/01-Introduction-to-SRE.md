# Chapter 1 — Introduction to SRE

## 1.1 Concept Explanation

Site Reliability Engineering (SRE) is the discipline of applying software engineering practices to operations. It was created at Google in 2003 by Ben Treynor Sloss, who described it as "what happens when you ask a software engineer to design an operations team." That sentence carries the whole philosophy: instead of running production with shell scripts, ad-hoc procedures, and heroic firefighting, you treat reliability as an engineering problem with code, measurements, and explicit tradeoffs.

The defining property of SRE is the **error budget**. A service does not need to be 100% reliable — it needs to be reliable enough that users do not notice. SRE quantifies "enough" with SLOs (Service Level Objectives), turning reliability from an opinion into a number. When the service is more reliable than its SLO, the team can ship features faster. When it is less reliable, the team must slow down and fix things. This single mechanism aligns product and operations incentives more cleanly than any other approach the industry has invented.

A second defining property: **toil reduction**. SREs do not accept that operational work must scale linearly with traffic. Every recurring manual task is a target for automation. If running the service requires more headcount as traffic grows, the team is failing.

## 1.2 Internal Working — How SRE Is Practiced

A mature SRE team works on five interlocking activities:

1. **Define and measure reliability** — SLIs, SLOs, error budgets, dashboards, and alerts.
2. **Manage incidents** — page, triage, mitigate, communicate, recover, postmortem.
3. **Reduce toil** — automate everything that repeats. Write tools, not tickets.
4. **Engineer reliability** — capacity planning, performance work, chaos engineering, resilience patterns.
5. **Partner with development** — pre-launch reviews, design consultations, production readiness gates.

The rhythm of the job is shaped by error budgets. When budgets are healthy, the team works on long-term improvements (the "fun" work — automation, tooling, platform). When budgets are exhausted, the team pivots to stabilization (the "necessary" work — bug fixes, capacity, rollouts of safer alternatives). This explicit pivot is unusual in operations roles, where work is usually whoever-shouts-loudest driven.

## 1.3 The Origin Story — Why Google Created SRE

In the early 2000s, Google was scaling faster than its sysadmin team could hire. The traditional model — separate dev and ops teams, dev throws code over the wall, ops keeps it running — broke down. Ops could not keep up; devs had no incentive to write reliable code; both sides resented each other.

Treynor's bet was: hire people with software engineering skills, give them operational responsibility, and let them solve operations problems with code. The result was SRE, which by 2016 was a documented discipline (in the "Google SRE Book," now free online) and by 2020 had been adopted across nearly every major tech company in some form.

The lesson generalized beyond Google: any organization running large-scale internet services benefits from treating reliability as a first-class engineering problem.

## 1.4 SRE vs DevOps vs Platform Engineering

These three roles overlap in confusing ways. The simplest distinctions:

| Role | Primary Focus | Primary Output |
|---|---|---|
| **DevOps** | Culture and process for fast, safe delivery | Pipelines, automation, shared responsibility |
| **SRE** | Reliability and operations as engineering | SLOs, runbooks, automation, postmortems |
| **Platform Engineering** | Internal developer platforms (IDPs) | Self-service tools that other teams use |

In practice these roles blur. A DevOps engineer at a startup may do all three jobs. An SRE at Google specializes in one service. A platform engineer at a mid-stage company builds the tooling that SREs and developers both use.

The honest description: SRE is a specific philosophy and methodology; DevOps is a culture; platform engineering is a product role aimed at internal developers. Many job listings mix the titles.

## 1.5 Real-World Use Cases

SREs own:
- Critical user-facing services (search, login, payments).
- Infrastructure platforms (Kubernetes clusters, observability stacks, CI/CD).
- Data plane reliability (databases, caches, message brokers).
- Networking and ingress.
- Capacity and cost.
- Incident response for company-wide outages.
- Production readiness reviews for new services.

SREs typically do not own:
- Feature development (though they consult heavily).
- Customer support escalations (though they triage technical ones).
- Vendor procurement (though they advise).

The line between "SRE owns it" and "developer owns it" is set by the organization. The healthiest pattern: services have an SRE liaison who works embedded with the dev team for a quarter, sets up SLOs and runbooks, and rotates back to the SRE team.

## 1.6 Production Architecture — Where SREs Operate

```
   Users
     |
   Edge (CDN, WAF, DNS)
     |
   Load balancer
     |
   Ingress / API gateway
     |
   Service mesh
     |
   +--- Services (owned by dev teams, partnered with SRE)
   +--- Data plane (databases, caches, queues — often SRE-owned)
   +--- Infrastructure (Kubernetes, cloud — SRE-owned)
     |
   Observability (SRE-owned)
     |
   Incident management & alerting (SRE-owned)
     |
   CI/CD platform (platform team or SRE)
```

SREs operate end-to-end but with depth in the lower layers. The mental model: developers care about features; SREs care about everything that breaks when features get popular.

## 1.7 Alternatives — When NOT to Use SRE

SRE is overkill for:
- Small teams with low traffic and forgiving users.
- Internal-only tools used by a handful of people.
- Early-stage prototypes still finding product-market fit.

For these, lighter operations work — a single ops-minded engineer, basic monitoring, and a manageable on-call — is sufficient. SRE practices add real overhead: SLO ceremony, postmortem discipline, dedicated toil reduction time. That overhead pays off above a certain scale and matters less below it.

## 1.8 Tradeoffs

Adopting SRE involves explicit tradeoffs:

| Decision | Win | Cost |
|---|---|---|
| Error budget governance | Aligns incentives | Slows feature work when exhausted |
| Dedicated SRE team | Specialization, focus | Coordination overhead with dev teams |
| Embedded SREs | Tight collaboration | Hard to scale, blurs roles |
| 100% SLA promises | Easy to sell | Impossible to meet, demoralizes team |
| Realistic SLOs | Sustainable | Customers may want higher numbers |
| Heavy automation upfront | Long-term productivity | Short-term feature slowdown |
| Comprehensive observability | Faster MTTR | Operational and cost burden |

The hardest tradeoff politically is the error budget. When the budget is exhausted, an SRE team must say no to feature launches. Without leadership backing, this rule collapses, and SRE becomes ops-with-a-fancy-name.

## 1.9 Scaling Challenges

SRE scales differently from feature engineering. Three patterns:

**Pattern 1 — One SRE per N developers.** Common target: 1:5 to 1:10 ratio. Below 1:10, SREs become bottlenecks. Above 1:5, SRE work duplicates effort.

**Pattern 2 — Platform SRE.** A central SRE team builds platforms used by many developers. The team is small but high-leverage. The risk: platform team loses touch with what dev teams actually need.

**Pattern 3 — Embedded SRE rotations.** SREs spend a quarter embedded with a dev team, then rotate. Knowledge transfer is high; expertise spreads.

At very large scale (Google, Meta, Amazon), SRE becomes a discipline with hundreds or thousands of practitioners, sub-specialties (storage SRE, network SRE, security SRE), and its own management hierarchy.

## 1.10 Security Concerns at the Introduction Level

SREs hold privileged credentials by necessity. They can read production data, modify infrastructure, and access systems other engineers cannot. This makes SRE practitioners high-value targets for attackers and high-trust positions internally.

Defensive practices:
- Just-in-time access (no permanent admin rights).
- Audit logs on every privileged action.
- Two-person rule for destructive operations.
- Mandatory training on insider risk and social engineering.

## 1.11 Deployment Guide — Starting an SRE Practice

A pragmatic six-month plan for a company adopting SRE:

1. **Month 1.** Hire or assign the first SRE. Establish SLO basics for one critical service. Set up baseline monitoring.
2. **Month 2.** Define SLIs and SLOs collaboratively with the dev team. Build dashboards.
3. **Month 3.** Establish on-call rotation, paging policy, incident roles.
4. **Month 4.** Write runbooks for top five alerts. Run a tabletop incident exercise.
5. **Month 5.** Introduce error budget policy formally. Discuss with leadership.
6. **Month 6.** Postmortem any incident. Identify the largest toil source. Begin automation work.

Resist the temptation to do everything at once. SRE done badly is worse than no SRE.

## 1.12 Monitoring Strategy

A first-pass SRE monitoring strategy covers four layers:

- **User-perceived metrics** — what users actually experience (page load time, error rate, feature success rate).
- **Service metrics** — RED method: Rate, Errors, Duration per service.
- **Infrastructure metrics** — USE method: Utilization, Saturation, Errors per resource (CPU, memory, disk, network).
- **Business metrics** — what executives care about (orders, signups, revenue) so reliability work ties back to outcomes.

Chapters 9-12 cover this in depth.

## 1.13 Cost Optimization — Introductory View

SRE owns reliability cost. Three principles:

- **Right-size.** Most fleets run at 20-40% utilization. Halving over-provisioned services usually saves more than rewriting any component.
- **Tier reliability.** Not every service deserves 99.99%. Tier-2 services at 99.9% are 10x cheaper.
- **Reserved and spot.** Steady-state on reserved instances, burst on spot.

Chapter 33 covers FinOps deeply.

## 1.14 Interview Perspective

Expect to be asked:
- *What is the error budget and how does it shape decisions?* The budget is `1 - SLO`. It governs how much risk you can take. When exhausted, feature work pauses.
- *Define SLI, SLO, SLA.* SLI is a measure; SLO is a target; SLA is a contract with consequences.
- *Walk me through a recent incident.* Be ready with detect → triage → mitigate → resolve → postmortem.
- *How is SRE different from DevOps?* DevOps is culture; SRE is a methodology with specific practices.

The interviewer is testing systems thinking, calm under pressure, and willingness to do unglamorous work. Memorize less; reason more.

## 1.15 Hands-on Exercises

1. Pick a public website. Define three SLIs you would track if you owned it. State an SLO for each.
2. List five tasks you do (or have seen done) that are pure toil. For each, sketch how you would automate it.
3. Draw the layered architecture in 1.6 for a product you know. Mark which layers an SRE would own.
4. Write a one-paragraph response to: "Our service hit its SLO last quarter. Why should we slow down feature work?"

## 1.16 Mini Project (Conceptual Design)

Design — on paper — an SRE setup for a fast-growing fintech with 50 engineers, 5 critical services, and one major outage per month. Specify:
- SRE team size and structure.
- SLOs for the top two services.
- Alerting strategy.
- On-call rotation shape.
- First three automation projects.
- First three reliability investments.

Return to this design after Chapters 5, 9, and 13.

## 1.17 Advanced Notes

- *The original Google SRE book is free online* (sre.google). The follow-up "Site Reliability Workbook" is the practical companion.
- *Modern SRE includes platform engineering elements* — internal developer platforms, golden paths, paved roads. The discipline has expanded.
- *Observability has shifted from "metrics + logs" to "metrics + logs + traces + events"*. OpenTelemetry standardized the wire format.
- *AI/ML systems require SRE specialization* — GPU scheduling, model drift, inference latency. Chapter 34 covers this.
- *Generative AI is now part of the SRE toolkit* — log analysis, runbook generation, code review for infra changes. Use it; do not trust it blindly.

## 1.18 Common Mistakes

- Treating SRE as a rename of operations without changing how work is done.
- Setting SLOs at aspiration ("five nines!") rather than reality.
- No error budget enforcement — the policy exists on paper but nothing changes when exhausted.
- One SRE for an entire org. The role demands specialization and team support.
- Hiring SREs without giving them ownership and authority over reliability decisions.
- Ignoring developer experience — making changes harder does not improve reliability long-term.

## 1.19 Enterprise Best Practices

Define SRE's charter explicitly. Get leadership commitment to error budget enforcement before claiming to do SRE. Pair SREs with development teams via embedded rotations or liaison models. Invest in observability and automation early — they are force multipliers. Hold regular "production review" meetings where every team reports on SLO status, incidents, and toil. Treat reliability as a product with roadmap and owners, not as a side effect of everyone trying their best.
