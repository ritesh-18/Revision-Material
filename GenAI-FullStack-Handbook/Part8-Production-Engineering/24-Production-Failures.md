# Chapter 24 — Production Failures

## 24.1 Concept Explanation

Every production GenAI system fails. The question is not whether but how often, how visibly, and how recoverably. This chapter catalogs the failure modes that recur across teams, the symptoms they produce, the root causes underneath, and the debugging approaches that work.

The mental model: a GenAI system is a stack of dependencies (LLM provider, vector DB, GPU pool, retrieval, application, network), each with its own failure profile, and the failure modes are dominated by the interactions between them.

## 24.2 OOM — Out of Memory

Symptom: pod crashes, "CUDA out of memory," API 500s. Often correlated with traffic spikes or specific request patterns.

Causes:
- KV cache exhaustion from too many concurrent users.
- A single request with extremely long context.
- Memory leak in long-running worker.
- Model loaded inefficiently (FP32 instead of BF16, wrong sharding).

Debug:
- Inspect GPU memory at the moment of crash.
- Identify the request that triggered it (logs).
- Cap max input tokens at the gateway.
- Lower max concurrent requests.
- Switch inference server to one with paged KV cache.

## 24.3 GPU Fragmentation

Symptom: GPU reports memory available but new requests fail to allocate.

Cause: KV cache allocations leaving holes too small for new sequences. Common in inference servers without paged attention.

Debug and fix:
- Use vLLM or another paged-attention server.
- Periodic restart of long-running inference pods.
- Lower max sequence length to bound fragmentation.

## 24.4 Memory Leaks

Symptom: memory usage grows over time even with steady traffic. Eventually OOM.

Causes:
- Conversation history accumulated without trimming.
- Cache without eviction policy.
- Reference cycles in async code.
- Tokenizer warmups loading new tokenizers per request.

Debug:
- Memory profiler over time (heaptrack, memray, py-spy).
- Diff heap snapshots before and after load.
- Check for unbounded collections in async tasks.

## 24.5 Deadlocks

Symptom: latency climbs to infinity; no obvious load.

Causes:
- Service A waits on B, B waits on A.
- Async tasks blocked on each other through shared mutex.
- Database connection pool exhausted; threads wait forever.
- Inference server waiting on KV cache pages that will never free.

Debug:
- Thread dumps.
- Distributed trace for stuck request.
- Look for circular waits.
- Check connection pool metrics.

## 24.6 Queue Overflow

Symptom: requests pile up, timeouts proliferate, no progress.

Causes:
- Burst traffic.
- Slow downstream service.
- Worker pool too small.
- Backpressure not propagating.

Debug:
- Queue depth metric over time.
- Identify slowest dependency.
- Add load shedding at the entry point.
- Scale workers up.
- Apply backpressure all the way to client.

## 24.7 Token Explosion

Symptom: cost spikes 5–50× overnight without traffic increase.

Causes:
- Retrieval starts returning more or larger chunks (a doc update or config change).
- A new feature adds long system prompt to every request.
- Agent loops fail to terminate, producing huge transcripts.
- A user submits very long inputs (intentional or accidental).

Debug:
- p99 input and output token counts.
- Distribution of tokens per request.
- Recent prompt and retrieval changes.
- Per-feature, per-user cost breakdown.

Fix:
- Per-request token cap.
- Per-user budget.
- Step limit on agents.
- Trim retrieval context before prompting.

## 24.8 Latency Spikes

Symptom: p99 latency jumps from 2s to 30s.

Causes:
- Downstream provider rate limiting.
- Cold starts.
- KV cache evictions causing recompute.
- Network blip to provider.
- Garbage collection in inference server.
- Long-tail prompts.

Debug:
- Per-request trace shows where time went.
- Correlate spikes with deploys, autoscaling events, provider incidents.
- Check provider status pages.

## 24.9 Vector DB Bottlenecks

Symptom: retrieval slow; QPS hits a ceiling.

Causes:
- Index too large for memory (HNSW).
- Filter selectivity poor.
- Hot shard from skewed tenant load.
- Replication lag.
- Rebuild in progress.

Debug:
- Vector DB internal metrics.
- Query plan inspection.
- Per-shard QPS.
- Replication status.

Fix:
- Resharding, more replicas, switch to a faster index, or use disk-resident at the cost of latency.

## 24.10 Model Drift

Symptom: quality scores degrade without code changes.

Causes:
- Provider updated their model under the same name.
- New user populations with different needs (data drift).
- Retrieval corpus changed.
- A dependency changed (tokenizer, embedding model).

Debug:
- Continuous eval over time.
- Diff against historical baseline.
- Identify when the regression started.

Fix:
- Pin model versions where providers allow.
- Re-evaluate on every dependency change.
- Update prompts to compensate.

## 24.11 Hallucination at Scale

Symptom: factual errors complaints increase.

Causes:
- Retrieval quality dropped.
- New types of questions not in eval set.
- Provider model change.
- Bad prompts encouraging speculation.

Debug:
- Sample bad outputs and inspect retrieved context.
- Compare against working historical examples.
- Run LLM-as-judge on production sample.

Fix:
- Strengthen retrieval.
- Tighten prompt with citation requirement.
- Refuse rather than guess when context is empty.
- Lower temperature.

## 24.12 Prompt Regression

Symptom: a new prompt deployed; quality drops on some queries.

Causes:
- Prompt change had unintended effect on edge cases.
- Eval set did not cover those cases.
- Provider's behavior changed how the prompt is interpreted.

Debug:
- Run new prompt against old eval set.
- A/B compare on production traffic.
- Add failing examples to eval set.

Fix:
- Roll back via feature flag.
- Patch prompt with explicit handling for failing case.

## 24.13 Agent Loops

Symptom: agent runs forever, costs spike, no result.

Causes:
- Tool returns ambiguous result; model keeps trying variants.
- Termination condition not detected by model.
- No step limit.

Debug:
- Inspect agent trace for repeated patterns.
- Check tool result format.

Fix:
- Hard step limit.
- Detect repeated tool calls with same args; break.
- Tighter tool descriptions and finalization instructions.

## 24.14 Cascading Failures

Symptom: one minor dependency fails; entire system goes down.

Cause: tightly coupled retries amplify load on the failing service, which causes its other consumers to slow, which causes those consumers' consumers to slow, until the whole system gridlocks.

Fix:
- Circuit breakers.
- Timeouts at every layer.
- Bulkheads to isolate failure domains.
- Graceful degradation.

## 24.15 Cold Starts

Symptom: first request after deploy or scale-up takes 30–300 seconds.

Cause: model weights must load into GPU memory.

Fix:
- Always-on warm replicas.
- Pre-load weights into image or local NVMe.
- Use scale-to-low rather than scale-to-zero for latency-sensitive paths.

## 24.16 Provider Outages

Symptom: every LLM request fails.

Cause: the provider is down.

Defense:
- Multiple providers behind an LLM gateway.
- Automatic failover with retries.
- Status page polling for awareness.
- Graceful degradation (cached responses, partial functionality).
- Subscribe to provider status notifications.

## 24.17 Rate Limit Whiplash

Symptom: large fraction of requests fail with 429.

Causes:
- Burst exceeded provider quota.
- Account-level limit hit.
- Per-key rate limit.

Defense:
- Distribute load across multiple API keys (where TOS allows).
- Implement client-side rate limiting that respects provider limits.
- Spread retries with jitter.
- Pre-arrange higher tier with provider.

## 24.18 Data Pipeline Stalls

Symptom: vector DB stops getting new documents.

Causes:
- Ingestion worker crashed and did not restart.
- Embedding API rate limited.
- Source connector broken.
- Dead-letter queue full.

Debug:
- Pipeline freshness metric.
- Worker logs.
- Source health.

## 24.19 Tenant Isolation Bugs

Symptom: one user sees another's data.

Causes:
- Cache keyed by prompt without tenant.
- Retrieval filter missing for one path.
- Session mixup.

Debug:
- Isolation test in CI.
- Audit retrieval and cache code paths.

Severity: this is a P0. Treat as security incident, not bug.

## 24.20 Bad Deploy Recovery

Symptom: a deploy regressed quality, latency, or cost.

Defense:
- Canary with auto-rollback on metric regression.
- Feature flags for fast disable without redeploy.
- Tagged immutable releases.
- One-button rollback.

## 24.21 Incident Response Workflow

```
   Detect (monitoring alert) -- minute 0
        |
   Triage (sev assessment, page on-call) -- minute 1-5
        |
   Contain (disable feature, roll back, kill switch) -- minute 5-30
        |
   Mitigate (apply patch or workaround) -- minute 30-120
        |
   Resolve (full fix deployed)
        |
   Postmortem (blameless, within 48 hours)
        |
   Action items (tracked to completion)
```

Without a documented runbook for the common failures above, every incident becomes a research project. Build the runbook over time; revise after each incident.

## 24.22 Debugging Approaches

- **Read the trace.** Distributed tracing shows where time went.
- **Inspect at the moment of failure.** Memory, GPU state, logs around the crash.
- **Bisect deploys.** Did the regression appear after a specific change?
- **Reproduce.** Try to reproduce the failing input in staging.
- **Diff configs.** Drift between environments is a frequent culprit.
- **Compare to baseline.** Same query yesterday vs today.

## 24.23 Tradeoffs

| Choice | Win | Cost |
|---|---|---|
| Aggressive retries | Mask transient failures | Amplify load during real outages |
| Hard timeouts | Bounded blast radius | Some requests truncated |
| Strict step limits | Cost bounded | Some legitimate agents stopped |
| Always-warm capacity | No cold start | Idle GPU cost |

## 24.24 Real-World Use Cases

- A 10× cost spike in a single day traced to a doc connector that started pulling images instead of text.
- A latency spike traced to a tokenizer warmup running per-request after an upgrade.
- A tenant leak traced to a global retrieval cache without per-user scoping.
- A quality regression traced to a provider model update on the same alias.

Each of these is real; each has happened to multiple teams.

## 24.25 Interview Questions

- Walk through a P0 incident from detect to postmortem.
- What is the most expensive failure mode you can think of for an agent?
- How do you detect tenant leakage?
- Design a kill switch architecture.
- Why is rolling forward sometimes worse than rolling back?

## 24.26 Hands-on Exercises

1. List the top 10 failure modes for your design from Chapter 1.
2. Write the runbook for each: detection signal, triage steps, mitigation, fix.
3. Plan a quarterly fire drill that exercises one runbook.

## 24.27 Common Mistakes

- No runbook; every incident is unique.
- Retries without circuit breakers.
- No load shedding; queues grow forever.
- Skipping postmortems; same incident recurs.
- Heroic fixes that bypass safety; the next failure is worse.

## 24.28 Enterprise Best Practices

Postmortems for every P0/P1, blameless. Runbooks for known failures. Quarterly fire drills. Tracked action items from every incident. SLOs that drive prioritization. On-call rotations with documented expectations. Treat reliability engineering as a permanent function.
