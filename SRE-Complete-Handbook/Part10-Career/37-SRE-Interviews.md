# Chapter 37 — SRE Interviews

## 37.1 Concept Explanation

SRE interviews differ from typical software engineering interviews. Algorithm questions still appear, but the loop emphasizes systems thinking, debugging under uncertainty, operational depth, and incident response. The goal: hire engineers who can keep production running, not just engineers who can write LeetCode solutions.

This chapter prepares you for SRE loops at companies from startups to FAANG. The exact format varies; the underlying skills do not.

## 37.2 The Typical Loop

A standard SRE loop:

1. **Recruiter screen** — background, motivation, role fit.
2. **Phone screen** — light coding, problem solving.
3. **System design** — design a reliable system at scale (1-2 rounds).
4. **Coding** — algorithms or scripting, less LeetCode-heavy than SWE.
5. **Troubleshooting** — debug a hypothetical incident.
6. **Operations / SRE depth** — SLOs, runbooks, on-call, postmortems.
7. **Behavioral / leadership** — past projects, culture fit, working with teams.

Senior loops add architecture interviews and may drop pure algorithm screens.

## 37.3 The Troubleshooting Interview

Unique to SRE loops. The interviewer presents a scenario: "Users are reporting the site is slow. Walk me through how you investigate."

What they look for:
- **Structured approach.** Detect → triage → investigate → mitigate → resolve → postmortem.
- **Hypotheses backed by data.** "I would look at p99 latency dashboards first."
- **Mental model of the stack.** Network, LB, app, DB, cache.
- **Mitigation before resolution.** "I'd roll back the recent deploy as a first mitigation."
- **Knowing when to escalate.**

Practice this with a friend. The first few times will be rocky.

## 37.4 The Operations Depth Interview

Questions about SRE practices specifically:

- *Walk me through how you set SLOs.*
- *What does a good runbook look like?*
- *Describe an incident you owned end-to-end.*
- *How do you reduce on-call burden?*
- *Compare blue-green and canary.*

The answers should come from real experience. If you've read this handbook, you can answer. If you have shipped production systems, your answers will have texture interviewers value.

## 37.5 The Coding Interview

SRE coding interviews are often:

- **Scripting.** Parse a log file. Count occurrences. Filter records.
- **Data structure use.** Implement a TTL cache, a rate limiter, a circular buffer.
- **Algorithm basics.** Strings, arrays, hashmaps. Less DP, less graph theory than SWE.
- **Practical.** Implement exponential backoff with jitter.

Some companies (Google, Meta) still ask harder algo questions. Most weight the system design more heavily.

Practice level: NeetCode 150 medium-tier is sufficient for most SRE roles.

## 37.6 The System Design Interview

Covered in depth in Chapter 36. Bring:
- Structured approach (requirements → HLD → component design → failure → scale → cost).
- Tradeoff vocabulary.
- Real examples from your experience.
- Familiarity with patterns in this handbook.

System design is where SRE seniors shine. Spend the most prep time here.

## 37.7 The Behavioral Interview

Have 8-10 STAR-format stories ready:

- An incident you handled.
- A tough call you made.
- A disagreement you resolved.
- A failure you owned.
- A win you delivered.
- A teammate you mentored.
- A process you improved.
- A risk you took.

Be specific. Numbers, names (where appropriate), outcomes. Avoid generic "we improved reliability" without details.

## 37.8 The Postmortem Discussion

Some loops include: "Tell me about a postmortem you wrote." Be ready with:
- The incident in 1-2 sentences.
- The root cause (systemic, not blameful).
- The action items.
- What you would do differently.

Demonstrating blameless postmortem skill is a strong signal.

## 37.9 The Take-Home

Some companies use take-home assignments instead of pure live coding:
- Build a small service with monitoring.
- Write a runbook for a given scenario.
- Design a system on paper.

These reveal real skill. Take them seriously; polish them; explain decisions.

## 37.10 Common Interview Questions — SRE Specific

A non-exhaustive list:

- *Define SLI, SLO, SLA, error budget.*
- *Explain how Linux load average works.*
- *Walk through what happens when you type a URL.*
- *How does TCP three-way handshake work?*
- *Difference between L4 and L7 load balancing.*
- *Explain Kubernetes pod lifecycle.*
- *What is the difference between liveness and readiness probes?*
- *Compare HPA, VPA, and Cluster Autoscaler.*
- *Explain CAP theorem.*
- *What is eventual consistency?*
- *Compare strong consistency and eventual consistency.*
- *What is a circuit breaker?*
- *Walk through cascading failure.*
- *How do you handle thundering herd?*
- *Compare blue-green and canary deployment.*
- *Explain GitOps.*
- *How do you debug high CPU on a server?*
- *Walk me through your observability stack.*
- *Compare Prometheus and Datadog.*
- *Explain a service mesh.*
- *What is mTLS?*
- *Walk through a Terraform workflow.*
- *What is drift in IaC?*
- *Compare reserved instances and spot.*
- *How do you reduce cloud cost?*
- *Walk through an incident.*
- *What's a blameless postmortem?*
- *How do you measure on-call health?*
- *Compare RPO and RTO.*
- *Design a multi-region failover.*
- *What is a chaos engineering experiment?*
- *Why is 100% availability the wrong target?*

Each should produce a 2-5 minute coherent answer.

## 37.11 Preparation Plan

Six weeks before interview:

**Week 1.** Read Chapters 1-8 of this handbook. Understand SLOs, error budgets, incident management.

**Week 2.** Chapters 9-16. Observability and reliability engineering.

**Week 3.** Chapters 17-24. Infrastructure and distributed systems.

**Week 4.** Chapters 25-36. Security, automation, AI, multi-region, system design.

**Week 5.** Practice system designs. Two per day, 30 minutes each, with someone listening.

**Week 6.** Mock interviews. Behavioral story rehearsal. Specific company research.

## 37.12 Mock Interviews

Tools:
- **Pramp** — free peer mock interviews.
- **Interviewing.io** — paid mocks with engineers from target companies.
- **Friend trades.** Most underrated.

Mocks reveal weaknesses no amount of solo prep will.

## 37.13 What Companies Look For

Across SRE roles:
- **Production experience.** Real systems, real failures, real fixes.
- **Calm under pressure.** Composed incident response.
- **Systems thinking.** Tradeoffs, dependencies, failure modes.
- **Communication.** Clear explanations, written and verbal.
- **Engineering mindset.** Automation first, not heroics.
- **Cultural fit.** Blameless, learning-oriented, team-first.

## 37.14 Junior vs Senior Loops

**Junior (0-3 years).** More coding, fundamentals, basic system design. Lean into learning and demonstrated curiosity.

**Mid (3-7 years).** Operational depth, ownership of incidents, real impact stories. System design at single-service scale.

**Senior (7+).** Cross-team initiatives, platform thinking, architecture across services. System design for the whole company.

**Staff+.** Strategic technical direction, organizational impact, technical influence.

Calibrate stories to level.

## 37.15 Compensation

SRE compensation is competitive with general software engineering — sometimes higher, especially at scale.

| Level | Base | Equity | Total |
|---|---|---|---|
| Junior | $130-170k | $20-50k | $150-220k |
| Mid | $170-220k | $50-150k | $220-370k |
| Senior | $220-300k | $100-300k | $320-600k |
| Staff | $300-400k | $300-700k | $600-1M+ |

(US ranges; international lower. Updated frequently — check levels.fyi.)

## 37.16 Negotiation

Standard practices:
- Know your market via levels.fyi.
- Have competing offers if possible.
- Negotiate equity refresh, not just initial grant.
- Sign-on bonus is easier to negotiate than base.

## 37.17 Target Companies for SRE

- **FAANG** — Google (where SRE was born), Meta, Amazon, Apple. Strong loops, strong comp.
- **Cloud providers** — AWS, Azure, GCP. Tons of SRE roles.
- **Infrastructure companies** — Datadog, HashiCorp, Cloudflare, Stripe, Snowflake, Databricks.
- **AI infrastructure** — OpenAI, Anthropic, Cohere, Together AI.
- **Enterprise SaaS** — Salesforce, ServiceNow, Workday.
- **Fintech** — Stripe, Square, Plaid, Robinhood.
- **Startups** — early SREs do platform building.

Each has slightly different culture and priorities.

## 37.18 The "Why SRE" Question

Common interview question. Have a real answer:
- "I love production. The systems I run touch users."
- "I'm energized by solving novel problems under uncertainty."
- "Engineering reliability into systems is my favorite work."
- "I see SRE as a force multiplier for engineering teams."

Generic answers (good salary, broad skills) sound rehearsed. Find your real motivation.

## 37.19 Common Mistakes

- Memorizing definitions without understanding.
- "I would use X" without explaining tradeoffs.
- Treating troubleshooting as algorithm puzzle.
- No stories ready for behavioral.
- Pretending to know what you don't.
- Not engaging with the interviewer's hints.

## 37.20 Hands-on Exercises

1. Write out answers to the 30 common questions in 37.10.
2. Practice system design out loud for one of the eight in Chapter 36.
3. Rehearse three behavioral stories with a friend.
4. Pick a target company; research it (recent blog posts, postmortems, public engineering practices).

## 37.21 Enterprise Best Practices (For Hiring Side)

For companies running SRE interviews:
- Standardized loops with calibrated questions.
- Diverse interviewer panels.
- Bar raisers / debrief discipline.
- Hire for trajectory, not pure pattern match.
- Onboarding plan that includes shadow on-call before solo.

## 37.22 Final Tips

- Read the GenAI handbook, this handbook, and a couple of public postmortems.
- Build something real you can talk about.
- Run mock interviews until you're bored.
- Sleep before the loop.
- Be yourself; SRE culture rewards humility and curiosity.

Good luck.
