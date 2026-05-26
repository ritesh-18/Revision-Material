# Chapter 7 — On-Call and Rotations

## 7.1 Concept Explanation

On-call is the practice of having engineers continuously available to respond to production issues. It is what makes 24/7 reliability possible. It is also the single biggest source of SRE burnout, and the area where small process improvements yield outsized quality-of-life gains.

A healthy on-call rotation feels manageable: a week of being on-call produces a small number of meaningful pages, each of which is actionable. An unhealthy rotation is multiple pages per night, repeated alerts for the same issue, and pages that no one can act on. The difference between healthy and unhealthy is engineering investment, not luck.

## 7.2 Why On-Call Exists

Three reasons society demands 24/7 availability:
1. **Global users.** Someone is always awake using your service.
2. **Asynchronous failure.** Systems break at random hours; you cannot wait until business hours.
3. **Time-to-action matters.** A 5-minute outage at 3 AM is much smaller than a 4-hour one.

Some systems do not need 24/7 on-call. Internal tools, batch pipelines, low-revenue services can have business-hours-only on-call with deferred response. Most user-facing services require always-on.

## 7.3 Rotation Models

**Follow-the-sun.** Teams in different time zones cover their daytime hours. No one is on-call at night. Best model when geographic distribution allows; rare in practice because few orgs have engineers in all required zones.

**Primary-and-secondary.** Two engineers on-call simultaneously. Primary takes the page; secondary backs up if primary is unreachable. Common in mid-sized teams.

**Weekly rotation.** Each engineer is on-call for one full week, then off for N weeks. Simple, predictable. The bias: pager fatigue late in the week.

**Daily rotation.** Engineer takes a 24-hour shift. Less fatigue per shift but constant context switching.

**Split shift.** Day shift (e.g., 9 AM–9 PM) and night shift (9 PM–9 AM) split between engineers. Good for distributed teams covering one geography from another.

**Manager backup.** A manager covers when no one else is available. Used rarely; signals that the team is understaffed.

The right model depends on team size, geography, page volume, and culture.

## 7.4 Healthy Rotation Math

Rules of thumb:

- **Minimum 6 engineers per rotation.** Below this, individuals are on-call more than 1 week per month, which is unsustainable.
- **Max 2 pages per shift.** Above this, on-call is exhausting and probably indicates alert quality problems.
- **No more than 25% of weekend nights paged.** Higher means weekends are not real off-time.
- **Hand-off meeting at shift change.** 15 minutes, what's hot, who's watching.

If the rotation cannot meet these, the team is understaffed, the alerts are too noisy, or both.

## 7.5 Paging Discipline

Not everything that fails should page a human at 3 AM. Three categories of operational signal:

**Page (urgent, wake-you-up):** customer-facing impact, SLO at risk, requires immediate action.

**Ticket (urgent but not 3-AM-urgent):** needs investigation soon but not now.

**Notification (informational):** chat or email, no action expected.

A common rule: if no one would act on it at 3 AM, do not page at 3 AM. Page is reserved for things requiring action *now*. Everything else flows to tickets or notifications.

Each page should have:
- A clear name.
- A runbook (Chapter 8).
- A confirmed action the responder will take.
- An SLO connection (why this page matters).

Pages without runbooks are a code smell. Either write the runbook or remove the page.

## 7.6 Alert Routing

A modern on-call setup routes alerts through tools like PagerDuty, Opsgenie, Splunk On-Call, Grafana OnCall, or incident.io.

Routing logic:
- Service identifies alert.
- Alert routes to on-call rotation for that service.
- Page rings primary's phone.
- If unacked in 5 minutes, page secondary.
- If unacked in 10, escalate to manager.
- If unacked in 15, escalate to director.

The escalation chain prevents pages from disappearing into the void.

## 7.7 Compensation

A real fact: on-call is work, and many teams treat it as work for pay purposes.

Common patterns:
- **On-call stipend** — flat per week of being on-call.
- **Per-page compensation** — small payment per actionable page.
- **Comp time** — day off after a heavy on-call shift.
- **Inclusive in salary** — no explicit pay (most common, fairest only with healthy page volume).

Different companies and regions handle this differently. EU labor law often requires explicit on-call compensation.

## 7.8 The Bad-On-Call Spiral

A failure mode:
1. Alerts are noisy.
2. On-call engineers burn out.
3. They start ignoring or muting alerts.
4. Real incidents get missed.
5. Leadership demands more alerting.
6. Alerts get noisier.
7. More engineers leave.

The way out: ruthless alert pruning, dedicated engineering time to improve alert quality, and management willingness to accept short-term alert blindness for long-term health.

## 7.9 The Good-On-Call Cycle

Inverse:
1. Each page is actionable, with a clear runbook.
2. On-call engineers complete shifts feeling competent.
3. Pages that turn out to be non-actionable are flagged and removed.
4. Engineers learn the system from each shift.
5. Alerting improves over time.
6. The rotation attracts engineers rather than driving them away.

Achieving this requires deliberate work.

## 7.10 Hand-off Practice

A weekly on-call hand-off meeting prevents context loss between shifts:

- What were the major pages this week?
- What was acked but not closed (still open)?
- Are any incidents ongoing?
- Are any planned changes happening (deploys, migrations, vendor maintenance)?
- Any known fragile parts of the system to watch?

15-30 minutes. Recorded for asynchronous follow-up.

## 7.11 On-Call Burden Tracking

Healthy teams track:
- Pages per shift (target: <3, ideally <1).
- Pages outside business hours.
- Hours of actual incident work per shift.
- Pages that were non-actionable (target: 0%, realistic: <10%).
- Repeat pages (same alert firing multiple times).

These metrics drive engineering investment. A team paged 10 times per shift should be doing nothing else until alert quality improves.

## 7.12 The Joel Test for On-Call

A quick health check. Answer yes/no:

- Do all pages have runbooks?
- Can every engineer in the rotation handle every page?
- Are pages acknowledged within 5 minutes on average?
- Are < 10% of pages non-actionable?
- Do you have at least 6 engineers per rotation?
- Is on-call discussed openly in team meetings?
- Are postmortems written for SEV1/2 incidents?
- Is there a compensation mechanism (stipend, comp time, or implicit)?
- Are weekends substantially quieter than weekdays?
- Do new joiners shadow for at least one shift?

Eight or more yes = healthy rotation. Five or fewer = the team should pause feature work and fix on-call.

## 7.13 Onboarding to On-Call

A new engineer should not be alone on-call until:
- They have shadowed 2-3 full shifts.
- They have handled at least 5 pages with backup.
- They know the runbook structure.
- They know the escalation chain.
- They have access to all required systems.
- They have practiced the most common incident types in a game day.

Pushing new engineers into on-call unprepared is irresponsible and produces bad outcomes for them and for users.

## 7.14 Tools

The on-call tooling ecosystem:

- **PagerDuty** — the incumbent. Mature, expensive, broad integrations.
- **Opsgenie** (Atlassian) — competitive feature set.
- **Grafana OnCall** — open-source, integrates with Grafana stack.
- **Splunk On-Call** (formerly VictorOps) — strong for Splunk customers.
- **incident.io, FireHydrant, Rootly** — newer, combine paging with incident response.
- **Squadcast** — value option.
- **xMatters** — enterprise.

Migration is painful; pick carefully. Mid-stage companies often start with PagerDuty and stick with it.

## 7.15 Real-World Use Cases

- A startup's on-call: 5 engineers, all sleeping with phones, no documented escalation. First major outage discovered the phone-on-silent problem. Fixed by adopting PagerDuty.
- A mid-stage SaaS had 30 pages per week per engineer. After a quarter of alert pruning and runbook investment, the number dropped to 5. Retention improved noticeably.
- A FAANG team has 20 SREs in three time zones doing follow-the-sun. No one is paged at night. Lots of investment to reach that state; it works.

## 7.16 Production Architecture

```
   Service emits alert
        |
   Alertmanager / monitoring tool fires
        |
   Routing rules send to on-call platform
        |
   Schedule determines current primary on-call
        |
   Primary paged (phone call, SMS, push, email)
        |
   Ack timer (5 min)
        |
   If unacked → secondary
        |
   If still unacked → manager → director → VP
        |
   Page acked
        |
   On-call begins triage (Chapter 5)
```

## 7.17 Tradeoffs

| Choice | Win | Cost |
|---|---|---|
| Small rotation (4 people) | Less hiring needed | Burnout fast |
| Large rotation (20 people) | Easy on each person | Context dilution |
| Generalist on-call | Anyone can fix anything | Slower learning |
| Specialist on-call | Faster fixes | Brittle if specialist sick |
| 24/7 same engineer | Continuity | Brutal shifts |
| Day/night split | Sleep preserved | More handoff |

## 7.18 Scaling Challenges

- One global service, multiple time-zone rotations need coordination.
- New services add to load — manage onboarding carefully.
- Org churn — losing senior engineers reduces depth.
- Pager creep — every new alert seems important; total grows.

## 7.19 Security Concerns

- On-call engineers have privileged access. Audit it.
- After-hours access requires special scrutiny.
- On-call calendars must be accurate (so the right person is responsible).

## 7.20 Deployment Guide

Standing up on-call from scratch:
1. List services and tier them.
2. Decide which need 24/7 on-call.
3. Recruit / assign 6+ engineers per service.
4. Define severity → action mapping.
5. Adopt a paging tool.
6. Build initial alert set (start conservative).
7. Write runbooks for each alert.
8. Train, shadow, then activate.
9. Iterate weekly on alert quality.

## 7.21 Monitoring Strategy

Metrics for on-call health:
- Pages per shift (count, distribution).
- Time-to-ack (median, p99).
- Non-actionable rate.
- Repeat rate.
- Weekend / off-hours fraction.

## 7.22 Cost Optimization

On-call has real cost: stipends, burnout-related attrition, engineering time spent on alerts instead of features. Reducing page volume is the cheapest optimization.

## 7.23 Interview Questions

- *Describe a healthy on-call rotation.*
- *How would you reduce page volume?*
- *Walk me through a tough on-call shift.*
- *What does an actionable alert look like?*
- *How do you onboard a new engineer to on-call?*

## 7.24 Hands-on Exercises

1. List your team's last 50 pages. How many were actionable?
2. Pick five non-actionable alerts. For each, decide: delete, downgrade to ticket, or fix the underlying issue.
3. Calculate the minimum rotation size for sustainable 24/7 coverage given your page volume.

## 7.25 Common Mistakes

- Page everything; sort it out later. (No: pages are expensive and degrade attention.)
- One person on-call for an entire org. (Lose them, lose coverage.)
- No runbooks. (Alerts trigger panic, not action.)
- No tracking. (Cannot improve unmeasured things.)
- Pretending on-call is not a workload. (Burnout follows.)

## 7.26 Enterprise Best Practices

Codify rotation policies. Track on-call metrics centrally. Compensate explicitly where labor law or culture demands. Invest in alert quality continuously. Require runbooks for every page. Run quarterly retrospectives on on-call health. Recognize on-call work in promotions and reviews — it is real engineering work that produces user value.
