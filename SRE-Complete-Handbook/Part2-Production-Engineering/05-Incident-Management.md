# Chapter 5 — Incident Management

## 5.1 Concept Explanation

Incident management is the structured process by which a team responds to a production problem, mitigates the impact, restores normal operation, and learns from the event. Without structure, incidents devolve into shouting, duplicate work, and missed mitigation. With structure, a team of strangers can resolve a serious outage in minutes.

Incident management has two sides: the **technical** (find and fix the bug) and the **coordination** (decide what to do, communicate with stakeholders, manage the response team). Many engineers focus on the technical and ignore the coordination. The coordination is what separates a 20-minute incident from a 4-hour one.

## 5.2 Internal Working — The Phases of an Incident

```
   Normal operation
        |
   Something breaks
        |
   Detection (alert fires)
        |
   Triage (severity assigned, roles assumed)
        |
   Investigation
        |
   Mitigation (stop the bleeding)
        |
   Resolution (fix the root cause)
        |
   Recovery (verify systems healthy)
        |
   Postmortem
        |
   Action items tracked to completion
```

Each phase has different goals and demands different behavior. Conflating them slows response.

## 5.3 Detection

The faster you detect, the smaller the impact. Detection comes from:
- Automated alerts (SLO burn, error rate, latency).
- Customer reports (slower than alerts; you do not want this to be your first signal).
- Internal employee reports.
- Third-party monitoring (status pages, synthetic checks).
- Social media (watch Twitter for outage reports).

Good alerting is so important it has its own chapter (Chapter 10).

## 5.4 Triage

Once an incident is declared, triage determines severity and assigns roles.

**Severity levels** (typical):
- **SEV1 / P0** — major user-facing outage, revenue impact. All hands.
- **SEV2 / P1** — degraded experience for many users.
- **SEV3 / P2** — partial impact or impact on small segment.
- **SEV4 / P3** — minor issue, address during business hours.

Severity drives response intensity, escalation, and external communication. Getting it wrong in either direction is costly — under-triage misses urgency, over-triage burns out the team on minor issues.

## 5.5 Incident Roles

For SEV1/SEV2, structured roles speed response:

**Incident Commander (IC)** — runs the incident. Does NOT debug. Coordinates, decides, delegates. The single person with authority to declare actions and escalations.

**Communications Lead (Comms)** — handles all external and internal communication. Updates the status page, internal Slack, executive briefings.

**Operations Lead (Ops)** — leads the technical investigation. Coordinates with subject-matter experts.

**Scribe** — keeps a timeline. Notes every action taken, every observation, every decision. Critical for postmortem.

**Subject-Matter Experts (SMEs)** — the people who actually debug. Pulled in as needed by Ops.

In smaller orgs, one person may wear multiple hats. The discipline is naming each role explicitly so nothing falls through.

## 5.6 The Incident Channel

When SEV1/SEV2 is declared:
- Create a dedicated channel (Slack, Teams) named `incident-YYYYMMDD-brief-name`.
- IC pins the current status to the channel.
- All discussion happens in the channel (not DMs).
- Every action is announced ("Restarting service X now").
- A separate channel may exist for executive/leadership updates.

The channel becomes the source of truth and the postmortem's primary input.

## 5.7 Investigation

Structured investigation beats chaotic poking. A useful framework:

1. **What changed recently?** Deploys, config, traffic, dependencies, external events.
2. **What is the blast radius?** Which users, which regions, which features?
3. **What does the data say?** Pull dashboards, logs, traces. Confirm hypotheses with data, not intuition.
4. **Form a hypothesis, test cheaply.** A quick test that disproves a hypothesis is better than a slow test that confirms it.
5. **Iterate.**

Beware of "the last change must be the cause" bias. Correlation is suggestive but not proof.

## 5.8 Mitigation Before Resolution

A core SRE mantra: **stop the bleeding first; understand later**.

Mitigation reduces user impact. Resolution fixes the root cause. They are different.

Mitigation tactics:
- Rollback the recent deploy (most common, fastest win).
- Feature flag the offending feature off.
- Failover to a healthy region.
- Restart the misbehaving service.
- Rate limit or shed load.
- Switch to degraded mode.
- Redirect traffic away from the affected component.

Resolution (root-cause fix) often takes hours or days and should not block mitigation.

## 5.9 The 5-Minute Rule

If a mitigation cannot be applied within 5 minutes, escalate or try a different mitigation. Long investigations without mitigation extend user impact.

## 5.10 Communication Discipline

During an incident, communication errors cause more damage than the technical issue.

**Internal communication:**
- Update the incident channel every 15-30 minutes even with "no new news."
- State current status, current actions, ETA if known.
- Avoid speculation. "We're investigating" is better than guessing.

**External communication:**
- Update the status page within minutes of confirmed user impact.
- Be honest about scope without overcommitting to ETAs.
- "Our team is actively investigating and we'll provide an update in 30 minutes" is fine.
- "We expect resolution in 10 minutes" is dangerous if you do not actually know.

**Executive communication:**
- One person (usually the IC or a designated comms lead) talks to executives.
- Executives are paying attention; they want updates but not technical details.
- Status, impact, what you're doing, ETA. That is enough.

## 5.11 Resolution and Recovery

Once mitigated, resolution work begins:
- Find the true root cause.
- Implement a fix.
- Test the fix.
- Deploy the fix.
- Verify the fix works in production.

Then recovery:
- Roll forward to normal capacity.
- Disable any temporary mitigations.
- Verify metrics return to baseline.
- Hold the incident channel open for an hour to catch regression.

## 5.12 Declaring Resolved

A formal close:
- IC announces resolution in the channel.
- Status page updated to operational.
- Postmortem scheduled (within 5 business days for SEV1/2).
- Action items captured immediately while memory is fresh.

## 5.13 Real-World Use Cases

A common SEV1 timeline:

- **00:00** Alert fires on 5xx spike.
- **00:02** On-call engineer acknowledges.
- **00:04** Confirmed customer-facing impact. SEV1 declared.
- **00:05** Incident channel created. IC assumes role. Comms updates status page.
- **00:08** Recent deploy identified as suspect. Rollback initiated.
- **00:12** Rollback complete. 5xx rate dropping.
- **00:15** Metrics normal. Incident channel holds for 1 hour.
- **00:30** Status page updated to operational. Channel kept open.
- **01:15** All-clear. Postmortem scheduled for next day.

Total user impact: ~12 minutes. Possible because the team knew the playbook.

## 5.14 Production Architecture

```
   Alert -> Paging (PagerDuty, Opsgenie)
        |
   On-call acks within 5 min
        |
   On-call decides severity
        |
   If SEV1/2: declare incident
        |
   Incident management tool (PagerDuty Incidents, FireHydrant,
   incident.io, Rootly, Jeli)
        |
   Roles assigned
        |
   Channel auto-created
        |
   Status page auto-updated (or manually)
        |
   War room (video + chat) established
        |
   Timeline auto-captured
        |
   Mitigation applied
        |
   Resolution + recovery
        |
   Postmortem template auto-generated
```

Modern incident response tools (incident.io, FireHydrant, Rootly, Grafana OnCall, PagerDuty Incidents) automate most ceremony — channel creation, role assignment, timeline capture, status page updates.

## 5.15 Alternatives

The role-based model (IC, Comms, Ops, Scribe) is FEMA's Incident Command System adapted for tech. It's the dominant approach.

Alternatives:
- Single responder for small teams (one engineer does everything).
- Two-person incidents (one debugs, one communicates) — common in mid-size orgs.

The full FEMA model becomes worth it at the scale where multiple SMEs may need to coordinate.

## 5.16 Tradeoffs

| Choice | Win | Cost |
|---|---|---|
| Formal IC role | Coordinated response | Ceremony for small incidents |
| Aggressive paging | Fast detection | Alert fatigue |
| Loose paging | Less fatigue | Slow detection |
| Always rollback first | Fast mitigation | Loses recent good changes too |
| Investigate before mitigate | Better understanding | Prolongs impact |
| Public status page | Customer trust | Pressure during incidents |

## 5.17 Scaling Challenges

- Multiple simultaneous incidents — need clear sub-IC structure.
- Time-zone gaps — handoffs between teams.
- Vendor outages — limited control, need workarounds.
- Cascading incidents — one fix triggers another problem.
- Long incidents (hours) — exhaustion, need rotation of IC.

## 5.18 Security Concerns

- Incident channels can leak sensitive data (customer info, internal architecture).
- Status page disclosures must be reviewed for legal/security implications.
- Some incidents (data breaches) require specific legal handling and law-enforcement notification.

## 5.19 Deployment Guide

Establishing incident management at a new team:
1. Define severity levels and what each triggers.
2. Define on-call rotation and paging policy.
3. Adopt an incident response tool (or use Slack templates).
4. Train everyone in IC and Comms roles.
5. Run a tabletop exercise (simulated incident) monthly.
6. Capture postmortems religiously.

## 5.20 Monitoring Strategy

Beyond service metrics, incident metrics matter:
- Mean Time To Detect (MTTD).
- Mean Time To Acknowledge (MTTA).
- Mean Time To Mitigate (MTTM).
- Mean Time To Resolve (MTTR).
- Incidents per week by severity.
- Repeat incidents (same root cause).

These guide where to invest improvement work.

## 5.21 Cost Optimization

Incidents are expensive: engineer time, customer credits, reputational damage. The economic case for incident management is strong. Tools like incident.io and FireHydrant pay for themselves quickly at any reasonable scale.

## 5.22 Interview Questions

- *Walk me through the last incident you handled.*
- *What is the difference between mitigation and resolution?*
- *Explain the role of incident commander.*
- *How do you communicate with executives during a SEV1?*
- *When would you declare a SEV1 vs SEV2?*

## 5.23 Hands-on Exercises

1. Draft a one-page incident playbook for your team: severity levels, on-call workflow, channel template, role definitions.
2. Run a tabletop: invent a scenario, have the team play through it for 30 minutes.
3. List your last three incidents. For each: MTTD, MTTM, MTTR. What was the slowest phase?

## 5.24 Common Mistakes

- IC also debugging — the IC must coordinate, not investigate.
- No scribe — no timeline for postmortem.
- "Quick chat" in DMs — fragmentation of state.
- Speculative external comms — promising ETAs that slip.
- Skipping the postmortem — incident recurs.
- One person knows the system; if they're asleep, the org is helpless.

## 5.25 Enterprise Best Practices

Standardize incident response across teams. Use a dedicated tool. Train everyone in IC. Hold quarterly drills. Track incident metrics company-wide. Make incident response part of new-hire training. Require postmortems for all SEV1/2 within 5 business days. Recognize good incident response publicly to build the muscle.
