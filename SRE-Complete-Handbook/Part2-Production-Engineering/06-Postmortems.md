# Chapter 6 — Postmortems and Blameless Culture

## 6.1 Concept Explanation

A postmortem is a written analysis of an incident, focused on what happened, why, and what will change so it does not happen the same way again. It is the institutional memory of an engineering organization.

The defining property is **blamelessness**. Postmortems do not name individuals as causes. They identify systemic conditions that allowed humans to make the mistakes they did. A blameless postmortem assumes that any engineer in the same position would have made the same error — and asks what about the system invites that error.

This is not "no accountability." Engineers are accountable for their work. But the postmortem is not the place for that conversation. The postmortem's purpose is learning, and learning requires safety to speak.

## 6.2 Why Blameless

The original argument from Sidney Dekker and others in safety science: blame stops learning. When people fear punishment, they hide information. Hidden information means systemic flaws stay hidden. Systemic flaws cause repeat incidents.

The blameless approach extracts more information. It assumes good intent and asks: given the information the responder had, the tools they had, the time pressure they were under, why did this action seem reasonable? That question reveals far more than "why did Bob screw up."

In practice: an engineer pushed a bad config that took down production. Blamed: Bob did a sloppy job. Blameless: the deploy pipeline accepted the config without validation, the staging environment did not exercise the affected code path, the rollback took 12 minutes because the procedure was untested. Three concrete systemic fixes versus one act of public shaming.

## 6.3 The Postmortem Structure

A standard postmortem template:

**Summary**
A short paragraph: what happened, when, who was affected, how long it lasted, severity.

**Impact**
- User-facing impact (errors, latency, missing features).
- Internal impact (teams blocked, builds broken).
- Business impact (revenue loss, customers affected, SLA credits).

**Timeline**
A chronological list with timestamps. Every detection, every action, every observation. The scribe's notes during the incident form this section.

**Root Cause Analysis**
The systemic explanation of why this happened. Not "Bob made a mistake" but "the config validation system had no schema for this field, the staging environment did not test config changes, and the change was deployed at 4 PM on a Friday."

**Contributing Factors**
Things that made the incident worse than it should have been: slow alerting, missing runbook, unclear ownership, fatigue from multiple incidents that week.

**What Went Well**
The detection was fast. The rollback worked. The communication was clear. Reinforce good behavior.

**What Went Poorly**
Honest list. Detection was slow. Rollback took longer than expected. Status page lagged.

**Lessons Learned**
The big-picture takeaways. What does this teach us about our systems and processes?

**Action Items**
Concrete, assigned, with deadlines. Each linked to a tracking ticket. Reviewed for completion in subsequent postmortem cycles.

## 6.4 The Five Whys

A simple but powerful root-cause technique. Ask "why" five times to drill from symptom to systemic cause.

Example:
- The site was down. Why?
- The database ran out of connections. Why?
- A new feature was opening connections without releasing them. Why?
- The library used in that feature does not release connections on certain error paths. Why?
- We did not catch this in code review. Why?
- We have no automated check for connection leaks. (Action item: add one.)

Stopping at "the developer wrote bad code" yields nothing. Continuing yields a real improvement.

The Five Whys is not the only technique (fishbone diagrams, fault trees, contributing factor analysis are alternatives), but it is the most accessible.

## 6.5 Identifying Action Items

Action items must be:
- **Specific.** "Improve monitoring" is not actionable. "Add p99 latency alert with burn rate threshold of 14.4 over 1h" is.
- **Assigned.** A person, not a team, owns it. Teams diffuse ownership.
- **Time-bound.** A target date. Even soft dates focus attention.
- **Tracked.** In a real ticket system, not in the postmortem doc.
- **Prioritized.** P0/P1 action items should be on the team's next sprint.

A postmortem with 20 vague action items is worse than one with 3 concrete ones. Focus on the changes most likely to prevent recurrence or reduce future impact.

## 6.6 Action Item Categories

A useful taxonomy:

- **Detection** — could we have detected this faster? (alerts, dashboards, monitoring).
- **Mitigation** — could we have mitigated faster? (runbooks, automation, rollback).
- **Prevention** — could we have prevented this entirely? (tests, code review, design changes).
- **Recovery** — could we have recovered faster? (validation, smoke tests, traffic shift).
- **Process** — were there process gaps? (training, documentation, on-call).

A balanced postmortem produces action items across multiple categories.

## 6.7 Real-World Use Cases — Famous Postmortems

Public postmortems worth reading:
- **AWS S3 outage, February 2017.** A typo in a command brought down a region. The investigation drove improvements in command tooling and blast radius limits.
- **GitHub October 2018 incident.** A 24-hour database degradation following a network partition. The postmortem details consistency, replication, and recovery in unusual depth.
- **Cloudflare July 2019 outage.** A regex change in a WAF rule consumed CPU on every server and brought down the network. Excellent case study in change risk.
- **Roblox October 2021 outage.** A 73-hour outage from a Consul service. Tour de force of debugging under pressure.
- **Atlassian April 2022 incident.** Two weeks of data loss for hundreds of customers from a buggy script. Long-running, complex postmortem.

Reading these teaches more about reliability than most courses.

## 6.8 The Postmortem Meeting

Not every postmortem requires a meeting, but SEV1/SEV2 usually benefit from one. Format:

1. **Pre-read.** The postmortem doc is shared 24 hours in advance. Attendees come having read it.
2. **Facilitator** (often a senior SRE) leads. Not necessarily the author.
3. **Walk through timeline.** Brief — focus on the moments where decisions were made.
4. **Discuss root cause.** Probe with questions. Surface assumptions.
5. **Refine action items.** Add, remove, sharpen. Ensure owners are present and agree.
6. **Capture additional observations.**

Meetings should be 60-90 minutes. Longer means the doc was not prepared.

## 6.9 The Postmortem Doc as Living Artifact

A postmortem is not done when published. It evolves:
- Action items get assigned and worked.
- Status updates added as items complete.
- Quarterly reviews check unresolved items.
- The doc is searchable and discoverable.

A "postmortem graveyard" of forgotten docs is worse than no postmortem at all.

## 6.10 Categorizing Incidents and Postmortems

A common practice is tagging postmortems for cross-incident learning:
- By cause (config error, code bug, capacity, dependency).
- By service.
- By severity.
- By detection mechanism.

Quarterly, review the patterns. If 40% of incidents are config errors, the next investment is config validation tooling.

## 6.11 Production Architecture for Postmortems

```
   Incident resolved
        |
   IC creates postmortem doc (from template)
        |
   Author (often on-call lead) drafts within 5 business days
        |
   Reviewers add comments
        |
   Postmortem meeting
        |
   Action items captured in ticket system
        |
   Publish to internal wiki
        |
   Tracked in central postmortem database
        |
   Quarterly review of action items
        |
   Annual review of patterns
```

Tools: Jeli (now PagerDuty Postmortems), Confluence templates, FireHydrant retrospectives, incident.io retros. Many teams use plain Google Docs or Markdown in Git.

## 6.12 Alternatives — Other Learning Practices

- **Operational reviews** — weekly meeting where teams report on incidents, SLOs, and toil.
- **Game days** — proactive practice incidents.
- **Pre-mortems** — before a launch, imagine it failed and ask why.
- **Postmortem trends** — quarterly cross-incident pattern analysis.

These complement postmortems but do not replace them.

## 6.13 Tradeoffs

| Choice | Win | Cost |
|---|---|---|
| Postmortem every SEV1/2 | Comprehensive learning | Author burden |
| Lightweight postmortems for SEV3 | Less burden | May miss learnings |
| Public postmortems (external) | Customer trust | Reveals architecture |
| Internal-only postmortems | Honest discussion | No external accountability |
| Strict blameless culture | Open discussion | Some prefer naming names |

## 6.14 Scaling Challenges

- High incident volume → not every incident gets a full postmortem.
- Distributed teams → asynchronous postmortem process needed.
- Action item completion rates drop over time without follow-up.
- Repeat incidents reveal that learning is not absorbed.

## 6.15 Security Concerns

- Postmortems contain sensitive info (architecture, customer data, vulnerabilities).
- Access controls on the postmortem store.
- Public postmortems must be redacted carefully.

## 6.16 Deployment Guide

Starting a postmortem practice:
1. Pick a template. Adapt from public examples (Google SRE workbook has good templates).
2. Train the team — what blameless means, why it matters.
3. Practice on a recent incident. Get reviews.
4. Establish the cadence — postmortem within 5 days, meeting within 10.
5. Track action items rigorously.
6. Quarterly review for trends.

## 6.17 Monitoring Strategy

- Postmortem completion rate (% of SEV1/2 with postmortems within target).
- Action item completion rate.
- Repeat incident rate (same root cause).
- Mean time to postmortem publication.

## 6.18 Cost Optimization

Postmortems are time-intensive. Reduce author burden:
- Pre-populate templates with timeline from incident tool.
- Auto-include affected metrics graphs.
- Co-authoring instead of solo authoring.

But never skip them for SEV1/2 to save time. The cost of skipping is higher (recurring incidents).

## 6.19 Interview Questions

- *What is a blameless postmortem and why?*
- *Walk me through a postmortem you wrote.*
- *How do you ensure action items get done?*
- *Talk about a postmortem where you disagreed with the conclusions.*
- *How do you handle a teammate who is defensive in a postmortem?*

## 6.20 Hands-on Exercises

1. Take a recent (or famous public) incident and write a blameless postmortem.
2. List 5 action items for it across detection, mitigation, prevention.
3. Practice rewriting blame-y language. "Bob deployed bad code" → "the deploy pipeline accepted code that lacked the validation we now know we needed."

## 6.21 Common Mistakes

- Blaming individuals (kills the practice).
- Vague action items (never get done).
- Postmortems without followup (recurring incidents).
- "Lessons learned" without changes (decoration, not improvement).
- Closing the doc and forgetting about it.
- Postmortems only for the worst incidents (small incidents teach too).

## 6.22 Enterprise Best Practices

Mandatory postmortems for SEV1/2 within fixed timeframes. Blameless culture trained and reinforced. Central postmortem repository with search. Quarterly trend analysis. Action items tracked to completion with monthly review. Postmortem skill development as part of senior engineer growth. Public postmortems for major external-facing incidents (builds customer trust).
