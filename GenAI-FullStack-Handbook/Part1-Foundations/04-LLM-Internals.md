# Chapter 4 — LLM Internals

## 4.1 Concept Explanation

A Large Language Model is, internally, a stack of transformer blocks plus an input embedding layer plus an output projection. Conceptually trivial. The reason LLMs are hard is everything *around* the model: tokenization, positional encoding, KV caching, sampling, batching, and quantization. This chapter explains what is happening inside the box while your application talks to it.

Understanding internals lets you reason about cost and latency without guessing. When a customer asks "why is the second user's response slower," you should be able to point at KV cache eviction. When the bill spikes, you should be able to point at the prompt that 10x'd input tokens via a long retrieval result.

## 4.2 Transformer Architecture Recap

A modern LLM is a *decoder-only* transformer. It is autoregressive — it predicts one token at a time, each new token conditioned on every previous token.

```
   Tokens ----> Embedding lookup ----> + Positional encoding
                                              |
                                       [Transformer Block 1]
                                              |
                                       [Transformer Block 2]
                                              |
                                             ...
                                       [Transformer Block N]
                                              |
                                       Final layer norm
                                              |
                                       Output projection (to vocab)
                                              |
                                       Logits over vocabulary
                                              |
                                       Sampling (temp/top-k/top-p)
                                              |
                                       Next token
```

Each transformer block contains multi-head self-attention followed by a feed-forward network, with residual connections and normalization. The block is repeated N times — 32 for a typical 7B model, 80 for a 70B, 96+ for the largest frontier models.

## 4.3 Attention Internals

Self-attention takes the sequence so far and, for each position, computes a weighted blend of all positions. The weighting comes from three learned projections:

- **Query (Q)** — what this position is looking for.
- **Key (K)** — what each position offers.
- **Value (V)** — the actual content each position contributes.

For each token, its Q is compared against every K via a dot product. The resulting scores are scaled, passed through softmax to become weights, and used to take a weighted sum of the V vectors. The output is a new representation of each position that incorporates information from the entire sequence.

The cost of full attention is *O(n²)* per layer. For 4,000 tokens, that is 16 million dot products per layer per head per sequence. For 100,000 tokens, that is 10 billion — and this is per layer, multiplied across dozens of layers. Long context is genuinely expensive.

## 4.4 Multi-Head Attention

Rather than computing one attention pattern, transformers compute many in parallel — typically 16, 32, or more — and concatenate the results. Each head can specialize. One might focus on syntactic relationships (subject-verb agreement), another on coreference (pronouns pointing to nouns earlier), another on positional patterns, another on factual recall.

You cannot directly inspect what each head is doing in production, but interpretability research has revealed reliable patterns. The operational lesson is that multi-head attention is what gives transformers their representational depth: many specialized lookups happening simultaneously.

## 4.5 KV Cache — The Optimization That Makes Generation Tractable

When a model generates a 1,000-token response, naïvely it would re-process the entire context plus all generated tokens for each new token — quadratic cost per token, cubic overall. KV caching prevents this.

The trick: for each token already processed, the K and V projections never change. So we cache them. When generating token N+1, we only compute Q for the new token and reuse all cached K and V vectors. This collapses per-token cost from O(n²) to O(n) and makes interactive generation feasible.

The cost: KV cache memory. For a 70B model with 80 layers and 8K context, the KV cache is roughly 20 GB *per user*. This is why batch sizes are limited even on big GPUs, and why paged attention (vLLM) was invented — to manage KV cache memory like virtual memory, allocating and freeing pages dynamically rather than reserving fixed slots.

## 4.6 Positional Encoding Internals

The transformer architecture itself does not know which token came first. Positional encoding is what gives it order. Three families matter today:

**Sinusoidal (absolute) encodings** — fixed sine and cosine waves of different frequencies added to embeddings. Used by the original transformer and BERT.

**Learned positional embeddings** — a separate embedding per position, learned during training. Used by early GPT models. Cannot extrapolate beyond training length.

**Rotary Position Embeddings (RoPE)** — rotates Q and K vectors in a position-dependent way before the dot product. The angle of rotation encodes position. RoPE generalizes better to longer contexts and is the standard in modern open models (Llama, Mistral, Qwen).

Operational consequence: when you see claims about "32K context window" or "128K context window," they almost always involve RoPE plus a scaling trick (NTK, YaRN, linear interpolation) to extend the model beyond its training length without full retraining.

## 4.7 Tokenization — Where Reality Meets the Model

Models do not see characters or words. They see *tokens*, which are sub-word fragments produced by a tokenizer. A token might be "the", "ing", " unhappiness", or "🦄". The tokenizer is a separate artifact trained once and frozen for the model's lifetime.

Why sub-word? Pure word-level tokenization explodes vocabulary size and chokes on rare or novel words. Pure character-level tokenization keeps vocabulary tiny but forces the model to process very long sequences. Sub-word splits the difference: common words become single tokens, rare or compound words split into pieces.

Implications you live with:
- "Token count" is what gets billed, not characters or words. English is roughly 0.75 tokens per word; code and non-English languages can be 2× or more tokens per word.
- Two different models likely have two different tokenizers, so the same text has different token counts and costs.
- Tokenizer quirks affect output. Trailing spaces, leading newlines, and unusual punctuation can fragment into multiple tokens and confuse the model.

## 4.8 BPE and SentencePiece — How Tokenizers Are Built

**Byte Pair Encoding (BPE)** starts with single bytes and iteratively merges the most frequent adjacent pair into a new token. After many merges, frequent fragments like "ing" or " the" become single tokens. GPT models and Llama use BPE variants.

**SentencePiece** is Google's library that supports BPE and a related algorithm called *unigram language model* tokenization. It treats text as raw bytes (no language-specific preprocessing), which makes it robust to any language. T5, Llama, and many multilingual models use SentencePiece.

The operational difference rarely matters at the application layer. What matters is consistency: never mix tokenizers between training and inference for the same model.

## 4.9 Embedding Models

Embedding models are transformer encoders (not decoders) trained to produce a single fixed-size vector for an entire input. The training objective is contrastive — pull semantically similar pairs together, push dissimilar ones apart.

Three families dominate open-source today: **BGE** (BAAI General Embeddings), **E5** (Microsoft), and **Instructor** (HKUNLP). Commercial options include OpenAI's text-embedding-3 series, Cohere's Embed v3, and Voyage AI's models.

Embedding models are typically 100M–1B parameters — much smaller than chat models — because the task is simpler. They run on a single small GPU at high throughput and dominate the cost of any large-scale RAG ingestion pipeline.

## 4.10 Decoder-only vs Encoder-Decoder

**Decoder-only** (GPT, Llama, Claude, Gemini in current configurations): one stack, causal attention, predicts next token autoregressively. Dominant for chat and code.

**Encoder-decoder** (T5, BART, original transformer): two stacks. Encoder reads the input bidirectionally, decoder generates output autoregressively while cross-attending to the encoder's output. Still strong for translation and summarization.

**Encoder-only** (BERT, RoBERTa, embedding models): one stack, bidirectional attention, no generation. Used for classification, embeddings, and retrieval.

In production GenAI today, decoder-only dominates for generation and encoder-only for retrieval. Encoder-decoder is more common in specialized tasks like translation pipelines.

## 4.11 Context Windows

The context window is the maximum number of tokens the model can attend to in a single forward pass. Typical values:

- 4K — older models like original GPT-3.5 and Llama 2.
- 8K to 32K — many current open models.
- 128K — current Claude, GPT-4, Llama 3.
- 200K to 1M+ — frontier models, often with sliding-window attention or other tricks.

A longer context is not free. Inference cost grows roughly linearly with input length thanks to KV caching, but memory grows linearly too — and there is a quality cost when models try to recall facts from deep within a long context (the "lost in the middle" effect). Larger context is a tool, not a default; in most production cases, retrieval (RAG) is cheaper and more accurate than stuffing.

## 4.12 Hallucination

A hallucination is a confident-sounding statement that is factually wrong. The model generates it because text-likelihood, not truth, is what it was trained to optimize. If the most likely-sounding continuation is wrong, the model says it.

Mitigations form a layered defense:
- Retrieval grounds the model in real documents and reduces invented facts.
- Citation prompts force the model to point at specific source passages.
- Lower temperature reduces creative drift.
- Tool use replaces model "knowledge" with API calls for facts (current time, latest price, recent events).
- Post-generation validation rejects outputs that fail format checks or cite non-existent sources.
- LLM-as-judge or human-in-the-loop verification catches what the prior layers miss.

No single mitigation eliminates hallucination. Production systems layer them.

## 4.13 Sampling Methods

**Greedy decoding** picks the most likely next token at each step. Deterministic, fast, but produces repetitive and dull text.

**Temperature sampling** scales the logits before softmax. Lower temperature = sharper distribution = more decisive choices. Higher = flatter = more diverse and risky.

**Top-k sampling** keeps only the k most likely tokens and renormalizes.

**Top-p (nucleus) sampling** keeps the smallest set of tokens whose cumulative probability is at least p. Adaptive — narrow when the model is confident, wide when uncertain. Most modern systems default to top-p around 0.9.

**Beam search** explores multiple candidate continuations in parallel and keeps the best. Good for tasks with a clear "right answer" like translation. Bad for open-ended generation; produces stilted, repetitive text. Largely replaced by top-p in modern chat.

**Speculative decoding** uses a small "draft" model to propose several tokens at a time, then verifies them in parallel with the big model. When the draft model agrees with the big model — which it does most of the time on easy tokens — generation speeds up substantially with no quality loss.

## 4.14 Quantization

Quantization reduces the numerical precision of model weights to save memory and increase throughput. Common levels:

- **FP16/BF16** — 16-bit. The native training precision for most modern LLMs.
- **INT8** — 8-bit integer. Roughly 2× memory savings versus FP16, minor quality loss.
- **INT4** — 4-bit integer. 4× memory savings versus FP16. Quality loss is noticeable but acceptable for many tasks. Enables running 70B models on consumer GPUs.
- **GGUF / GPTQ / AWQ** — different quantization formats with different tradeoffs. GGUF dominates llama.cpp ecosystems. AWQ and GPTQ dominate GPU inference servers.

Quantization is lossy. Always benchmark the quantized model on your tasks before assuming "INT4 is fine."

## 4.15 Inference Pipeline End-to-End

```
   User input
        |
   Tokenizer  -->  token IDs
        |
   Prefill phase: process all input tokens in parallel,
   compute K and V for every layer, store in KV cache
        |
   Decode phase: one token at a time
        For each new token:
          - Run Q through cached K/V
          - Compute logits
          - Sample
          - Append token to KV cache
        |
   Detokenizer -->  user-visible text
        |
   Stream tokens to client
```

The prefill phase is parallel and compute-bound. The decode phase is sequential and memory-bandwidth-bound. This split is why production inference servers like vLLM treat them as different scheduling problems.

## 4.16 GPU Memory Flow During Inference

```
   GPU VRAM at idle:
      [ Model weights (e.g., 14 GB for 7B FP16) ]
      [ Empty space ]

   GPU VRAM during one user's request:
      [ Model weights ]
      [ KV cache for this user (grows per token) ]
      [ Activations (small, transient) ]

   GPU VRAM during 16 concurrent users:
      [ Model weights (shared) ]
      [ KV cache user 1 ]
      [ KV cache user 2 ]
      [ ...                       ]
      [ KV cache user 16 ]
```

The model weights are *shared* across all concurrent users — batching is essentially free from a memory perspective. The KV cache, however, is *per-user* and is what limits concurrency. Paged attention (vLLM) treats KV cache as virtual memory pages so the GPU can pack more users in by avoiding fragmentation.

## 4.17 Tradeoffs

| Choice | Win | Cost |
|---|---|---|
| Longer context | More information available to the model | Quadratic compute, linear memory, "lost in the middle" |
| Lower temperature | More predictable, factual outputs | Less creative, occasional repetition |
| Quantized weights | Cheaper inference, smaller GPUs | Some quality loss, more complex tooling |
| Bigger model | Better reasoning, broader knowledge | Higher cost, slower, harder to host |
| Smaller model + RAG + tools | Cheap and accurate | More moving parts, more engineering |

## 4.18 Common Mistakes

- Assuming output tokens are billed the same as input tokens (output is usually more expensive).
- Forgetting that system prompts are tokens too and inflate every request.
- Using greedy decoding and being surprised by repetition.
- Treating quantization as free without measuring downstream quality.
- Ignoring the difference between prefill and decode latency in SLAs.

## 4.19 Interview Questions

- Walk through the inference pipeline end to end.
- Why is KV cache memory the limit on concurrent users?
- Explain RoPE and why it enables long context.
- Speculative decoding — why does it work without quality loss?
- Compare INT8 and INT4 quantization tradeoffs.

## 4.20 Hands-on Exercises

1. For a 13B model with 40 layers, head dim 128, 40 heads, and an 8K context, estimate KV cache size per user in FP16.
2. Take a paragraph of your own writing. Count words. Use an online tokenizer for any popular model to count tokens. Compute the ratio.
3. Sketch where prefill ends and decode begins in a chat interface that streams responses.

## 4.21 Enterprise Best Practices

Pick the smallest model that meets quality and benchmark it on your actual tasks, not generic leaderboards. Standardize the tokenizer used for billing and context budgeting. Track prefill latency and decode latency separately; they have different optimization paths. Monitor KV cache hit rate and concurrent slot usage in production inference clusters. Prefer top-p around 0.9 with temperature 0.2–0.7 as a safe default for factual chat workloads, and only deviate with measurement.
