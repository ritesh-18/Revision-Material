# Chapter 39 — Building SRE Teams

## 39.1 Concept Explanation

This chapter is for people responsible for forming, growing, or managing SRE teams. Whether you are a first-time hiring manager, a founder hiring your first SRE, or a director building an SRE function, the patterns matter.

A well-built SRE team multiplies engineering productivity. A poorly built one becomes the team that gets paged at 3 AM and resents it. The difference is intentional design.

## 39.2 When to Hire Your First SRE

Signals it's time:
- Production is breaking weekly.
- Engineers spend more than 20% of time on operations.
- Major incidents lack postmortems.
- Capacity planning is "we'll add servers when it breaks."
- Onboarding new services takes weeks.
- No one is comfortable being on-call alone.

Too early: 5-engineer startup running on Heroku. They don't need an SRE; they need to ship product.

Too late: a 100-engineer company with no SRE function. Recoverable but painful.

The sweet spot: 30-50 engineers, real production, growth ahead.

## 39.3 First SRE Hire Profile

Look for:
- 5+ years experience.
- Worked at companies one stage further than yours.
- Coding capability (not just ops).
- Has built tools others used.
- Calm in incident discussion.
- Comfortable with ambiguity.

Avoid:
- Pure sysadmin without software skills.
- Specialist in one narrow tool.
- Heroic firefighter who doesn't automate.
- Anti-developer attitude.

The first hire sets the tone. Take time to find the right fit.

## 39.4 Team Structures

Multiple SRE patterns exist:

**Embedded.** SREs sit inside product teams. Tight collaboration; shared knowledge. Risk: dilutes SRE community and practices.

**Central.** All SREs on one team, supporting many product teams. Strong community; risk of being seen as "service" team.

**Mixed.** Central team for platform; embedded SREs in product. Common at scale.

**Liaison.** Each product team has an SRE liaison who rotates back to central team periodically. Combines benefits.

**Platform engineering.** Build internal platform; product teams use it. Increasingly common.

No single right answer. Choose by org size, culture, and product complexity.

## 39.5 Defining the Charter

Critical first step: what does the SRE team own?

Options:
- All production operations.
- Reliability, capacity, observability platform.
- Critical services only.
- Platform and toolchain.

Write it down. Get leadership sign-off. Revisit annually.

Without a clear charter, SRE becomes "anyone's problem we get blamed for."

## 39.6 Error Budget Authority

The SRE team must have authority to enforce error budgets. Without leadership commitment, the policy is theater.

Concretely:
- When budget is exhausted, SRE can pause feature launches.
- VP/director must back this.
- Product agrees in advance.
- Disagreements escalate to documented authority.

Discuss this before accepting the SRE charter. If error budget enforcement won't be supported, the SRE practice will degrade.

## 39.7 Hiring Strategy

For a team of N SREs:

- **1st hire.** Senior, broad. Sets practices.
- **2nd-3rd.** Senior + mid. Build the team.
- **4th-5th.** Add specialists (observability lead, platform lead).
- **6th-10th.** Round out coverage. Add juniors.
- **10+.** Sub-team specialization.

Maintain 1 SRE per 5-10 developers as a rough ratio.

## 39.8 The Interview Loop You Run

Mirror the SRE interview practices (Chapter 37):
- Phone screen with the hiring manager.
- Coding (scripting / tools, not LeetCode-heavy).
- System design (production-focused).
- Troubleshooting.
- Operations / SRE depth.
- Behavioral.

Calibrate panel across interviews. Debrief discipline.

## 39.9 Onboarding

First 30 days:
- Read service architecture docs.
- Pair with on-call (no solo).
- Read recent postmortems.
- Run game days as observer.
- Build one small tool or fix.

30-90 days:
- Solo on-call shadow → solo on-call.
- Own one project.
- Write a postmortem.
- Contribute to runbooks.

Onboarding well retains people. Throwing new hires into pager rotation week one is how you lose them.

## 39.10 Culture

SRE culture you should aim for:

- **Blameless.** Mistakes are systemic.
- **Documentation.** Verbal knowledge is not knowledge.
- **Automation.** Toil is the enemy.
- **Calm under pressure.** Incidents are routine.
- **Curiosity.** "Why" until satisfied.
- **Generosity.** Help other teams.
- **Pride in operations.** The work is meaningful.

Culture is set by what you reward, what you tolerate, and what you ignore. Be intentional.

## 39.11 On-Call Health

A first responsibility: protect on-call health.

- Page count tracked weekly.
- Alert quality improvements prioritized.
- New on-callers shadow first.
- No one solo for the first month.
- Comp time / stipend per company policy.
- Rotation feedback collected.

A burning team is a leaving team.

## 39.12 SLOs as a Practice

Implement SLOs for top services first. Don't try to SLO everything at once.

Steps:
1. Pick a Tier 0/1 service.
2. Define SLIs.
3. Measure baseline.
4. Set SLO slightly tighter than baseline.
5. Implement burn rate alerts.
6. Write error budget policy.
7. Get leadership sign-off.
8. Roll out and review monthly.
9. Add next service.

Slow rollout beats big-bang.

## 39.13 Postmortem Culture

Mandatory practice from day one:
- SEV1/2 incidents get postmortems within 5 days.
- Blameless.
- Action items tracked.
- Templates standardized.
- Quarterly trend reviews.

The team's reaction to mistakes signals safety. Punish mistakes; learning stops. Encourage learning; culture compounds.

## 39.14 Internal Communication

The SRE team needs to be visible:
- Weekly newsletter or status email.
- Monthly all-hands update.
- Regular postmortem readouts.
- Public SLO dashboards.

Visible work is funded work. Invisible work gets cut.

## 39.15 Working with Developers

SRE-developer relationship is the most important relationship in the org.

Health signals:
- Devs include SRE in design reviews.
- SREs are invited to product planning.
- Devs respect on-call constraints.
- SREs respect feature priorities.

Unhealthy signals:
- "It works on my machine" thrown over wall.
- SRE only consulted in crisis.
- Devs ignore reliability work.
- SREs ignore product needs.

Investment in the relationship pays back continuously.

## 39.16 Growing the Function

As the company grows:
- Add specialists (network, observability, platform).
- Form sub-teams.
- Hire platform engineers separately or merge.
- Add tech leads.
- Add an engineering manager (or two).

The right structure shifts with size. Stay nimble.

## 39.17 Promoting SRE Work

SREs are sometimes invisible. The work that prevents incidents is the work no one notices. Mitigate:

- **Metrics visible.** SLO compliance, MTTR, toil hours.
- **Cost savings tracked.** Dollar figures resonate.
- **Wins celebrated.** Reliability improvements; bad incidents prevented.
- **Promotion criteria** that value reliability work as much as feature work.

Without this, SRE careers stall at a level lower than feature engineers.

## 39.18 The Anti-Patterns

Failure modes for SRE teams:

- **Sysadmin renamed.** Same work, no engineering rigor.
- **Toil sink.** All time spent on ops; no improvements.
- **Adversarial.** Devs vs SREs.
- **Permission gate.** SRE blocks everything; nothing ships.
- **Single hero.** One person knows everything; they leave.
- **Cargo cult.** Adopt Google's practices without context.

Each is real. Each has killed SRE teams.

## 39.19 Investing in the Team

Things that help SRE teams thrive:
- Training budget (conferences, courses).
- Books and learning materials.
- Hackathon time for tool building.
- Cross-training opportunities.
- Recognition.
- Reasonable on-call.
- Career growth conversations.

Engineers who feel invested in stay.

## 39.20 Measuring Team Health

Quarterly review:
- SLO compliance.
- Incident metrics (frequency, severity, MTTR).
- On-call burden per engineer.
- Toil % vs engineering %.
- Retention.
- Engagement.
- Project completion rate.

If multiple are red, intervene.

## 39.21 Real-World Use Cases

- A startup hired its first SRE at 30 engineers. Within 6 months: SLOs, runbooks, healthy on-call. Engineering velocity went up because nothing was on fire.
- A mid-stage company had no SRE; CTO did it. Burnout, incidents. Hired an SRE lead; within a year the function existed and the CTO got their job back.
- A FAANG company has thousands of SREs. Sub-teams by service. Distinguished engineers in the function. Career path through and through.

## 39.22 Common Mistakes

- Hiring a junior as the first SRE (no one to learn from).
- No charter (everything is SRE's job, so nothing is).
- No leadership backing for error budget enforcement.
- Pushing junior into solo on-call.
- Hiring SRE as gatekeeper (devs resent).
- No engineering time; all toil.

## 39.23 Enterprise Best Practices

Documented charter, signed by VP. Defined relationship to product teams. Clear on-call expectations. Standardized practices across all teams. Career path documented. Annual planning aligned with engineering org. Internal community (Slack channels, lunches, meetups).
