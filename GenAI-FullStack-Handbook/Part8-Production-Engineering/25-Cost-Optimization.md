# Chapter 25 — Cost Optimization

## 25.1 Concept Explanation

A GenAI bill grows in three dimensions: the number of users, the amount each user does, and the cost per unit of work. The job of cost optimization is to compress the third dimension without harming the first two. Done well, a 50% cost reduction is routine; 80% is achievable on many workloads. Done poorly, cost optimizations degrade quality, and users disappear.

The discipline is twofold: measure what costs money, and apply the cheapest mitigation first.

## 25.2 Where the Money Goes

For most GenAI products, costs sort roughly as:

1. **LLM inference** (50-80% of the bill).
2. **Vector DB and embeddings** (10-20%).
3. **GPU infrastructure** (overlaps with #1 for self-hosted).
4. **Cloud overhead** (storage, networking, observability) (5-15%).
5. **Third-party AI services** (reranking, STT/TTS, image, etc.) (variable).

The hot path is LLM inference. Most levers below target it.

## 25.3 Model Choice

The single biggest cost lever. A frontier model can be 50–500× the price of a small model. Many tasks do not need frontier capability.

The discipline:
- Benchmark your task on multiple models.
- Define a quality threshold; pick the cheapest model that clears it.
- Re-benchmark quarterly as new models release.

A typical production architecture uses two or three model tiers: small/cheap for the bulk of requests, medium for harder ones, frontier for the hardest few percent.

## 25.4 Model Routing

A router classifies incoming requests by difficulty and sends each to the appropriate model tier.

Router signals:
- Input length and complexity.
- Detected intent.
- Required output structure.
- Historical performance per query type.

A good router cuts costs 50-70% with negligible quality loss. A bad router shifts hard queries to weak models and tanks quality. Measure carefully.

## 25.5 Caching

The cheapest token is the one you do not generate. Three cache tiers:

**Exact prompt cache.** Hash the full prompt; if seen, return cached output. Works for FAQ-style or batch workloads.

**Semantic cache.** Embed the prompt; if a semantically similar prompt was recently answered, return that. Works for paraphrased queries. Risk: false positives — make sure the cached answer is general enough.

**Prefix cache.** Most LLM providers now offer "prompt caching" that bills repeated prefixes at a discount. System prompts and few-shot examples become cheap when stable.

Combined, caches typically eliminate 30–60% of LLM calls on chat workloads.

## 25.6 Inference Batching

Self-hosted inference benefits enormously from batching. A single request might use 5% of a GPU's throughput; 32 batched requests use 80% with similar per-request latency. This is the central insight behind vLLM.

For API-based providers, batching is invisible to you, but they pass much of the cost savings on via their own pricing.

## 25.7 Quantization

INT8 saves 2× memory; INT4 saves 4×. Smaller memory = more concurrent users per GPU = lower cost per request.

Always benchmark quality. Some tasks tolerate INT4 with no measurable loss; some show clear degradation. The right answer is task-specific.

## 25.8 Speculative Decoding

A small draft model proposes tokens; the large target model verifies. When agreement is high (which it usually is), generation speeds up 2–5× with no quality loss.

Implementation effort is non-trivial but the savings are real for high-volume workloads.

## 25.9 Context Compression

Long prompts cost more. Compression techniques:

- **Truncate retrieval** to the most relevant chunks.
- **Summarize conversation history** when it grows long.
- **Compress repeated instructions** into shorter forms.
- **Drop low-value context** (turn-by-turn metadata that the model can infer).

A 30% prompt reduction is a 30% input cost reduction.

## 25.10 Output Length Control

Output tokens cost more than input tokens (typically 2–4×). Control output length:

- **Explicit max_tokens** limits.
- **Prompts that ask for concise answers** when concise is appropriate.
- **Structured outputs** that bound the output shape.
- **Skip "explain your reasoning"** when it is not needed.

## 25.11 Smaller Embedding Models

Embedding cost scales with token count and model size. A smaller embedding model (384-dim instead of 1536-dim) cuts embedding cost and vector storage simultaneously.

Matryoshka embeddings (truncatable) let you use a 3072-dim model trained for high quality, store 256 dims for low cost, and switch dynamically per query.

## 25.12 Self-Hosting Thresholds

API calls are cheaper at low volume; self-hosting is cheaper at high volume. The crossover depends on:
- Model size (smaller models cross over earlier).
- Utilization (idle GPUs are pure cost).
- Engineering cost of running inference infra.
- Negotiated provider discounts.

Rough heuristic: at 1M tokens/day on a small model, API is fine. At 10M tokens/day on a small model, consider self-hosted. At 100M tokens/day, self-hosted almost certainly wins on unit cost — if you can keep GPUs utilized.

## 25.13 Spot and Preemptible GPUs

Cloud spot/preemptible GPUs cost 50–90% less than on-demand. Suitable for:
- Batch inference (eval runs, embeddings).
- Stateless workloads with fast restart.
- Resilient services with replica diversity.

Not suitable for latency-critical chat that cannot tolerate cold restarts.

## 25.14 Reserved Capacity

For steady-state workloads, commit to 1- or 3-year reserved capacity. Savings of 30–60% versus on-demand. The risk: you pay even if usage drops.

Match reservation to your forecast lower bound; use on-demand for the variable component above.

## 25.15 Right-Sizing

The most common waste: GPUs too big for the model, replicas too many for the load.

- Profile actual GPU utilization. Many production fleets run at 20–40%.
- Bin-pack small models on shared GPUs via MIG.
- Reduce replica counts where latency allows.
- Match GPU class to workload (L40S over H100 when L40S is sufficient).

## 25.16 Network Cost

Cloud egress is expensive. For AI:
- Keep inference and storage in the same region.
- Use private connections (PrivateLink) for cross-cloud or hybrid.
- Compress payloads (gzip, brotli) on the wire.
- CDN for static assets.

Many AI products discover hidden egress costs when looking at monthly bills.

## 25.17 Vector DB Cost

Levers:
- Quantization (PQ, scalar, binary) for storage.
- Lower-dim embeddings.
- Tier old data to cold storage.
- Right-size managed tier.
- Self-host above a threshold.

For large corpora, vector DB cost can exceed inference cost. Audit it.

## 25.18 Eval Cost

Continuous evals can run up costs. Mitigations:
- Sample production traffic for continuous eval.
- Run heavy evals nightly, lightweight per-PR.
- Use cheaper judge models with periodic agreement checks vs frontier judges.

## 25.19 Cost Accounting

You cannot optimize what you cannot measure. For each request, track:
- Tokens in, tokens out, by model.
- Cost in USD.
- Feature / endpoint.
- User / tenant.

Aggregate to dashboards:
- Top features by cost.
- Top users by cost.
- Cost trend by day, by week.
- Cost per resolved task — the business metric.

Set anomaly alerts on cost; sudden spike = leak.

## 25.20 Cost-Quality-Latency Triangle

You can usually get two of the three. Make the tradeoff explicit per feature.

| Feature class | Optimize for |
|---|---|
| Real-time chat | Latency + quality, accept higher cost |
| Background batch | Cost + quality, accept higher latency |
| Internal tool | Cost + latency, accept lower quality |

## 25.21 Worked Example

A chat product:
- 100k MAU, 5 turns/day, 30 days/month = 15M turns/month.
- Average tokens per turn: 2,000 in, 500 out.
- Frontier model: $5/M input, $20/M output. Per turn: $0.01 + $0.01 = $0.02. Monthly: $300,000.
- Add prefix cache (system prompt stable): 30% saving on input → $240,000.
- Route 70% of easy queries to a cheaper model (10× cheaper): $96,000.
- Add semantic cache for 20% deduplication: $77,000.
- Trim retrieval context by 30%: $65,000.

From $300k to $65k. None of those changes degrade quality if done carefully.

## 25.22 Real-World Use Cases

- A startup cut inference bill 80% by routing easy queries to a fine-tuned 7B model.
- An enterprise cut vector DB bill 70% by switching to binary embeddings and reranking with full-precision.
- A consumer app cut latency and cost simultaneously by adopting speculative decoding.
- A SaaS company cut egress cost 60% by colocating inference and storage in one region.

## 25.23 Tradeoffs

| Optimization | Save | Risk |
|---|---|---|
| Cheaper model | 5-100× | Quality loss |
| Caching | 30-60% | Stale answers |
| Quantization | 2-4× memory | Quality loss |
| Self-hosting | 30-70% | Ops burden |
| Spot capacity | 50-90% | Interruption |
| Reserved | 30-60% | Commitment risk |

## 25.24 Interview Questions

- Walk through cost optimization for a $1M/month chat product.
- When does self-hosting beat API pricing?
- Design a model router and explain its failure modes.
- How do you measure cost per resolved task?
- Compare semantic caching and prefix caching.

## 25.25 Hands-on Exercises

1. Build a cost calculator for your design from Chapter 1.
2. Identify the top three levers for your workload and estimate savings.
3. Plan a quarterly cost review process.

## 25.26 Common Mistakes

- Optimizing the wrong layer (saving pennies on storage while inference burns dollars).
- No per-feature cost visibility.
- Skipping caching as "too complex."
- Routing without measuring; degrades quality silently.
- No cost alerts; finding a 10× spike at month-end.

## 25.27 Enterprise Best Practices

Cost dashboards per feature, per tenant. Monthly reviews. Budgets per team. Anomaly alerts. Quarterly model and embedding reviews as new options release. Treat cost as a quality dimension equal to latency and accuracy.
