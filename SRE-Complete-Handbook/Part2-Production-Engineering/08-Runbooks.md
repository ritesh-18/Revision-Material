# Chapter 8 — Runbooks and Playbooks

## 8.1 Concept Explanation

A runbook is the documented procedure for handling a specific operational situation. When the database connection pool exhaustion alert fires at 3 AM, the on-call engineer reads the runbook and acts. The runbook is what turns "we know we have a problem" into "we are now fixing it" in seconds rather than hours.

The terminology varies:
- **Runbook** — step-by-step procedure for a known scenario.
- **Playbook** — broader procedure for a class of incident, with decision branches.
- **SOP (Standard Operating Procedure)** — formal name in some industries.
- **Wiki article** — informal name.

In this chapter we use "runbook" for any actionable, repeatable procedure.

The defining property: a runbook is **followed**, not read. It should be possible to execute it without thinking deeply. The thinking happened when the runbook was written.

## 8.2 Why Runbooks Matter

Three benefits:

**Speed.** A tired on-call engineer at 3 AM cannot solve novel problems efficiently. Runbooks turn novel into routine.

**Consistency.** Different engineers handle the same problem the same way. Reduces variance in response quality.

**Knowledge transfer.** Senior engineer knowledge becomes a shared asset. The senior can sleep; the runbook holds the knowledge.

The cost of not having runbooks: every page becomes a debugging session, every shift requires the most senior person, and on-call burnout follows.

## 8.3 What Makes a Good Runbook

A runbook is good if it can be followed by a tired engineer at 3 AM, who has never seen this exact issue before, without paging someone more senior.

Specifically, a good runbook:
- Names the alert or scenario clearly.
- States the impact.
- Lists prerequisites (access, tools).
- Provides verification steps so the engineer can confirm the issue.
- Provides mitigation steps in order.
- Lists what NOT to do.
- Has escalation criteria.
- Links to related resources.

A bad runbook is "the alert means X; investigate." That is no help.

## 8.4 The Runbook Template

A practical template:

**Alert Name** — exact name as it appears in the paging system.

**Severity** — what severity this maps to.

**Impact** — what users experience when this fires.

**Likely Causes** — short list of common causes, ordered by frequency.

**Verification Steps** — how to confirm the issue is real and not a false alarm. Include the exact dashboard links, log queries, or commands.

**Mitigation Steps** — numbered list. Each step:
- The action to take.
- The expected result.
- What to do if it does not work.

**Escalation** — when to escalate. Who to escalate to.

**Communication** — what to tell stakeholders. Templates for status page updates.

**Related Runbooks** — links to runbooks for related issues.

**Last Reviewed** — date, by whom.

## 8.5 Examples of Runbook Content

A snippet for a hypothetical "DatabaseConnectionPoolExhausted" alert:

---

*Alert: DatabaseConnectionPoolExhausted*

*Severity: SEV2 (immediate user impact, errors visible on login flow)*

*Impact: User login fails with 503. Affects new sessions; existing sessions unaffected.*

*Likely Causes (most common first):*
1. *Connection leak in recently deployed code.*
2. *Long-running query holding connections.*
3. *Underlying DB instance is slow, queueing requests.*
4. *Increased traffic exceeding configured pool size.*

*Verification:*
- *Open the [DB Connections dashboard]. Confirm `active_connections` is at or near max.*
- *Check [recent deploys dashboard] for changes in the last 4 hours.*
- *Run `SHOW PROCESSLIST` on the primary DB. Look for connections in `Sleep` for >60s.*

*Mitigation (try in order):*

1. *If there was a deploy in the last 4 hours, roll it back.*
   - *Command: `kubectl rollout undo deployment/api -n prod`*
   - *Expected: pool drains within 60s.*
   - *If pool does not drain, proceed to step 2.*

2. *Kill long-running queries.*
   - *Query: see [DBA scratchpad / kill query template].*
   - *Expected: pool frees up within 30s.*
   - *If unchanged, proceed.*

3. *Increase pool size temporarily.*
   - *Action: edit [config map name], increase from 100 to 200.*
   - *Apply: `kubectl apply -f ...`*
   - *Note: this is a temporary fix; do not leave at 200 longer than 1 hour without consultation.*
   - *Escalate to DBA team if you need to go higher.*

*Escalation:*
- *If mitigation fails after 15 minutes, escalate to DBA on-call.*
- *If user impact exceeds 30 minutes, escalate to incident commander.*

*Communication:*
- *Status page template: "We are investigating login issues affecting a subset of users."*

*Related Runbooks:*
- *[SlowQueryAlert]*
- *[DatabaseFailover]*

*Last reviewed: 2026-04-15 by Sarah*

---

This level of specificity is what makes a runbook actually useful at 3 AM.

## 8.6 Where Runbooks Live

Options:
- **Confluence / Notion** — searchable, easy to edit. Common.
- **Git-backed (markdown in a repo)** — versioned, reviewed via PR. Better for code-adjacent teams.
- **Linked from the alert itself** — every page contains the runbook URL. This is critical regardless of where the runbook lives.

Whatever the tool, two requirements:
- **Discoverable.** The on-call engineer should find the runbook in <30 seconds.
- **Editable.** When the runbook is wrong, the next responder can fix it immediately.

## 8.7 Runbook Lifecycle

Runbooks rot. Systems change; runbooks do not auto-update. Lifecycle practices:

- **Review on use.** When you follow a runbook during an incident, note what was wrong or outdated. Fix it before closing the incident.
- **Quarterly review.** Schedule each runbook for a quarterly check. Touch it or retire it.
- **Last-reviewed field.** Display prominently. Auto-flag runbooks older than 6 months.
- **Owner per runbook.** A specific person responsible for accuracy.

The biggest risk is the runbook that worked in 2023 and has not been touched since.

## 8.8 Pre-Filled Commands and Templates

For high-stakes mitigations, pre-fill the exact commands. The on-call engineer should not be inventing kubectl invocations under pressure.

Better: include a one-liner the responder can copy-paste. Include "dry-run" variants. Mark destructive commands clearly.

Even better: build a CLI tool that wraps the mitigation. `oncall fix db-pool-exhaustion --service api` is safer than a five-line kubectl recipe.

## 8.9 Decision Trees

For complex scenarios with branches, runbooks use decision trees:

```
   Alert: HighLatency
        |
   Q: Is the latency spike on one endpoint or all?
        |
   ===> One endpoint
        |
        Q: Was there a recent deploy touching that endpoint?
            ===> Yes → Roll back, follow Runbook A.
            ===> No → Check downstream dependencies, follow Runbook B.
        |
   ===> All endpoints
        |
        Q: Is GC time elevated?
            ===> Yes → Memory pressure, follow Runbook C.
            ===> No → Check infrastructure, follow Runbook D.
```

Trees keep simple cases simple while supporting complex ones.

## 8.10 Game Days for Runbook Testing

A runbook is hypothetically useful. A runbook that has been executed in practice is verified useful. **Game days** test runbooks by triggering simulated incidents and watching responders follow them.

A typical game day:
1. Schedule a 90-minute window.
2. Inject a fault (kill a pod, exhaust connections, fail a dependency).
3. Real on-call responds using runbooks.
4. Observers note where runbooks helped, where they fell short.
5. After the exercise: revise runbooks based on findings.

Game days reveal that the team's confidence in their runbooks is often misplaced. The first time you run one is usually humbling.

## 8.11 Automated Runbooks

The endpoint of runbook maturity: automation. Instead of a human reading and executing steps, a tool does it.

Examples:
- An alert auto-runs a Lambda that performs the mitigation.
- A ChatOps bot accepts `/runbook fix-db-pool` and runs the steps in production with audit logging.
- A Kubernetes operator detects the condition and applies the fix.

Automated runbooks reduce MTTR dramatically. They also raise the stakes — a bad automated runbook can cause outages. Build them carefully, test them, include circuit breakers.

## 8.12 Real-World Use Cases

- A team's most-paged alert had no runbook for a year. Six engineers each independently figured out the same fix and ran it differently. Once one engineer documented the runbook, the next page resolved in 4 minutes instead of 40.
- A company built a self-service ChatOps tool that exposed runbooks as commands. On-call engineers could trigger common mitigations from Slack. Average MTTR dropped 60%.
- A fintech automated their database failover runbook. Manual failover took 25 minutes; the automated version takes 3 with no on-call action. Net effect: customers do not notice failovers anymore.

## 8.13 Production Architecture

```
   Alert configured with runbook URL
        |
   Page fires, on-call sees URL in notification
        |
   Click to runbook (wiki, git repo, ChatOps)
        |
   Engineer follows steps
        |
   Optional: ChatOps wraps steps as commands
        |
   Optional: Automation runs the steps directly
        |
   Incident resolved
        |
   Runbook updated based on what worked / didn't
```

## 8.14 Tradeoffs

| Choice | Win | Cost |
|---|---|---|
| Detailed runbooks | Fast response | Author effort |
| Brief runbooks | Low maintenance | Engineers must improvise |
| Wiki-based | Easy to edit | Versioning weak |
| Git-based | Versioned, reviewed | Higher friction |
| Manual runbooks | Flexible | Requires humans |
| Automated runbooks | Fast MTTR | Risk of bad automation |

## 8.15 Scaling Challenges

- Runbook sprawl: hundreds of runbooks, no one knows which is current.
- Drift: runbooks fall out of sync with systems.
- Knowledge silos: runbooks for legacy systems only one person can update.
- Searchability: finding the right runbook in seconds.

## 8.16 Security Concerns

- Runbooks contain operational secrets (commands, paths, sometimes credentials).
- Access controls on the runbook store.
- Audit logging on runbook execution (especially automated).

## 8.17 Deployment Guide

Building runbook practice:
1. List the top 20 alerts by frequency.
2. Write a runbook for each.
3. Link runbook URL from the alert configuration.
4. Train the team on the format.
5. Establish review cadence.
6. Game day the top runbooks.
7. Begin automating the highest-frequency ones.

## 8.18 Monitoring Strategy

- Runbook coverage (% of alerts with runbooks).
- Runbook age (median, max).
- Runbook usage (clicked through during incidents).
- Time-to-mitigate by runbook (do runbooks help?).

## 8.19 Cost Optimization

A runbook saves 5-30 minutes per incident on average. At $200/hour engineer cost and 10 incidents per month, even a modest runbook investment pays back in weeks.

## 8.20 Interview Questions

- *What makes a good runbook?*
- *Walk me through a runbook you authored.*
- *How do you keep runbooks from rotting?*
- *When should a runbook be automated?*
- *Describe a game day you ran.*

## 8.21 Hands-on Exercises

1. Take your team's most frequent page. Write a runbook for it using the template in 8.4.
2. Pick a runbook (any team's). Identify three improvements.
3. Design a game day for one alert. List the steps.

## 8.22 Common Mistakes

- Vague "investigate the issue" runbooks.
- No runbook for high-frequency pages.
- Runbooks that have not been reviewed in a year.
- Hidden runbooks no one can find.
- Runbooks that point to other runbooks that point to other runbooks (circular).
- Skipping the verification step ("are we sure this is actually the issue?").

## 8.23 Enterprise Best Practices

Every page has a runbook. Runbooks are reviewed quarterly. A runbook owner is assigned per runbook. Game days are run monthly. Automation targets the top 10 highest-frequency runbooks. Runbook quality is part of production readiness reviews. The team tracks "runbook-resolved" vs "novel debug" ratio as a maturity signal.
