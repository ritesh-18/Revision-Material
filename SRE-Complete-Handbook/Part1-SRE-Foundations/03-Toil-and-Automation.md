# Chapter 3 — Toil and Automation

## 3.1 Concept Explanation

**Toil** is the kind of work that grows linearly with the size of the service: manual, repetitive, automatable, reactive, devoid of enduring value. Restarting a stuck process, rotating a certificate, manually scaling a deployment, copy-pasting from a runbook — all toil.

The Google SRE book defines toil with six attributes. Work is toil if it is:
1. **Manual** — done by hand.
2. **Repetitive** — happens more than once.
3. **Automatable** — a machine could do it.
4. **Tactical** — interrupt-driven, not strategic.
5. **Without enduring value** — once done, no lasting improvement.
6. **Scales linearly** — twice the traffic means twice the work.

The SRE imperative: cap toil at 50% of an SRE's time. The other 50% must be engineering work — automation, tooling, reliability improvements. Without this cap, SREs become senior sysadmins. With it, they become force multipliers.

## 3.2 Why Toil Matters

Toil is not just unpleasant — it is dangerous. Three failure modes:

**Career stagnation.** Engineers who spend their days clicking the same buttons stop learning. They become unhireable elsewhere because their skills are not transferable.

**Burnout.** Repetitive interruptions destroy focus and morale. Engineers burn out fast on toil, much faster than on hard but novel work.

**Unbounded growth.** If toil is not capped, every traffic increase means more headcount. The business cannot scale economically.

Capping toil forces investment in automation and tooling that scale sublinearly.

## 3.3 Identifying Toil

A weekly toil log is the standard practice:

- At end of each week, every SRE lists tasks performed.
- Each task is tagged "toil" or "engineering."
- Hours are summed.
- Quarterly review: if any individual is above 50% toil, this is a P0 problem.

Common toil examples:
- Restarting hung services.
- Manually responding to alerts with known fixes.
- Bumping resource limits on noisy pods.
- Rotating secrets.
- Granting access requests.
- Adding new tenants manually.
- Updating dashboards by hand.
- Triaging known low-severity alerts.

Hidden toil to watch for:
- Reviewing the same PR types ("update the version number").
- Mentoring on the same questions repeatedly.
- Filling out the same forms (incident reports, access requests).
- Following the same runbook steps.

## 3.4 The Toil-Eradication Method

A four-step loop:

**Step 1 — Measure.** Track toil weekly. Build the toil log.

**Step 2 — Rank.** Sort by total hours, frequency, and urgency. The 80/20 rule applies: 20% of toil categories produce 80% of the hours.

**Step 3 — Eliminate or Automate.** For the top item:
- Can it be eliminated entirely? (Sometimes the work is unnecessary.)
- Can it be self-served? (Move the action to a portal the requester uses.)
- Can it be automated? (Write code that does it.)
- Can it be reduced? (Find the root cause and prevent recurrence.)

**Step 4 — Verify.** A month later, did toil decrease? If not, what went wrong?

Repeat against the new top item.

## 3.5 What Is NOT Toil

Distinguishing toil from valuable operational work matters.

**Not toil:**
- Designing a new automation system (engineering work).
- Investigating a novel incident (learning work).
- Writing a postmortem (institutional knowledge).
- On-call standby time (necessary, not toil per se).
- Code review of teammates' changes (collaboration).
- Capacity planning analysis (strategic).

The test: if the work is novel, strategic, or produces lasting artifacts, it is not toil even if it feels tedious.

## 3.6 Automation Categories

Automation appears on a spectrum:

**Level 0 — No automation.** Human runs each step manually.

**Level 1 — Documented runbook.** Steps are written down. Humans follow.

**Level 2 — Scripted.** A shell or Python script automates the steps. Human runs the script.

**Level 3 — Self-service.** A web UI or CLI tool lets non-experts trigger the action.

**Level 4 — Event-driven.** Automation runs in response to an event (alert, schedule, webhook).

**Level 5 — Fully autonomous.** System detects, decides, and acts without human input.

Most SRE work targets Level 3-4. Level 5 is rare and reserved for high-confidence remediations (auto-scaling, auto-failover, auto-restart of crashed pods).

## 3.7 The Automation Hierarchy

Not all automations deserve equal investment:

- **Self-healing** (Level 5) — high value. Eliminates pages.
- **Self-service** (Level 3) — high value. Eliminates tickets.
- **Scripts** (Level 2) — medium value. Speeds up humans but still requires them.
- **Runbooks** (Level 1) — baseline. Every alert needs one but it's not automation per se.

The pattern: invest in moving items up the hierarchy, starting from the top of the toil list.

## 3.8 Self-Service Tooling

The single highest-leverage SRE work is building self-service tools that other engineers use.

Examples:
- A portal where devs request new database instances (instead of filing tickets).
- A CLI that creates new services from a template (with monitoring, alerts, deployment wired up).
- A web UI for granting time-bound access to production resources.
- A button that safely restarts a specific service with rollback.
- A dashboard generator that creates standard dashboards for any new service.

Self-service tools are platform work. They scale: one tool serves hundreds of engineers and saves SRE time forever.

## 3.9 Closed-Loop Automation

The most powerful pattern: automation that **detects, decides, and acts** without human input.

Examples:
- HPA (Horizontal Pod Autoscaler) scales pods based on metrics.
- Kubernetes restarts crashed pods.
- Cluster autoscaler adds nodes when pods cannot schedule.
- Auto-failover for databases when primary fails.
- Auto-rollback when error rate spikes after a deploy.

The pattern requires three components:
- **Sensor** — measurement of state.
- **Controller** — logic that decides action.
- **Actuator** — the thing that performs the action.

Closed-loop automations need careful design. A wrongly-tuned auto-remediation can cause more outages than it prevents (cascading restarts, flapping autoscaling). Always include circuit breakers and human override.

## 3.10 Toil Budget

A formal practice at some companies: track toil hours per engineer per quarter. Above a threshold, leadership intervenes. Below, the team is rewarded.

This makes toil a managed budget rather than an invisible drain.

## 3.11 Real-World Use Cases

- A company spent 40 SRE-hours per week on manual cert rotation. Built a tool to automate it. Now spends 1 hour per month verifying.
- A team got paged 30 times per week for "memory high" on a service. Built auto-scaling and the page stopped. The team shipped its quarterly roadmap for the first time in a year.
- A platform team built a "new service" CLI that creates the repo, deploys to staging, sets up monitoring, and adds the service to the portal in 5 minutes. Saved hundreds of hours per quarter across the org.

## 3.12 Production Architecture

A toil-reduction program at scale:

```
   Weekly toil log (per SRE)
        |
   Aggregation dashboard
        |
   Quarterly toil review
        |
   Top toil items
        |
   Automation work assigned (engineering project)
        |
   Self-service tool / closed-loop automation
        |
   Toil reduces; new top items emerge
        |
   Cycle continues
```

## 3.13 Tradeoffs

| Choice | Win | Cost |
|---|---|---|
| Cap toil at 50% | Sustainable, scalable | May feel slower short-term |
| Automate first | Long-term productivity | Upfront investment |
| Tolerate toil short-term | Faster firefighting | Slow death spiral |
| Self-service tooling | High leverage | Tooling team needed |
| Closed-loop automation | Eliminates pages | Risk of bad automation |

## 3.14 Scaling Challenges

- Tooling has its own toil (maintenance).
- Self-service tools must be discoverable; otherwise engineers default to tickets.
- Closed-loop automation needs careful testing.
- Toil migrates — automating one source may surface another.

## 3.15 Security Concerns

Automated systems with high privileges are high-value attack targets. Defenses:
- Least privilege for automation accounts.
- Audit logs on every automated action.
- Rate limiting on destructive operations.
- Kill switches for runaway automation.

## 3.16 Deployment Guide

Implementing toil reduction at a new team:

1. **Establish baseline.** Have everyone log toil for two weeks.
2. **Identify the top three sources.**
3. **Pick the most automatable.**
4. **Allocate dedicated engineering time** (often 20% of each engineer's week).
5. **Build, ship, measure.**
6. **Repeat.**

## 3.17 Monitoring Strategy

- Toil hours per engineer per week.
- Toil as percentage of total work.
- Automation coverage (% of alerts that have auto-remediation).
- Self-service tool usage (how many tickets did the portal replace).
- MTTR trends (good automation should reduce these).

## 3.18 Cost Optimization

Automation often pays for itself many times over. Track ROI:
- Engineer hours saved per month.
- At company loaded cost ($150-300/hr), automation paying back in months is typical.

## 3.19 Interview Questions

- *What is toil and how do you measure it?*
- *Walk me through how you would reduce toil for a team drowning in pages.*
- *Difference between self-service tooling and closed-loop automation.*
- *When is automation a bad idea?*

## 3.20 Hands-on Exercises

1. List the last 10 manual tasks you (or a team you know) performed. Tag each as toil or not.
2. Pick one. Sketch a self-service tool that would eliminate it.
3. Estimate hours saved per year if the tool succeeded.

## 3.21 Common Mistakes

- Treating "I'm too busy to automate" as a real reason. Automation is the way out.
- Building tools no one uses (poor UX, undiscoverable).
- Closed-loop automation without circuit breakers (flapping, cascading failures).
- Automating bad processes instead of redesigning them.
- Tooling team that builds for itself, not for users.

## 3.22 Enterprise Best Practices

Formalize toil tracking. Allocate 20-30% of team time for engineering work explicitly. Reward automation work in performance reviews. Maintain a "self-service catalog" so engineers know what tools exist. Invest in a platform team whose job is reducing toil across the company. Treat tooling like a product — with users, roadmap, and ownership.
