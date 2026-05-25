# Chapter 8 — Embeddings

## 8.1 Concept Explanation

An embedding is a fixed-size vector that represents the meaning of a piece of content — a word, sentence, paragraph, image, or audio clip. Two pieces of content that mean similar things end up with vectors that point in similar directions. This single property turns search from keyword matching into semantic understanding.

Without embeddings, "how do I reset my password" matches the documentation page titled "Password Reset Procedure" only if the keywords overlap. With embeddings, the two are recognized as semantically equivalent regardless of wording. Every retrieval-augmented system, semantic search engine, and recommendation system in the modern stack rests on this.

## 8.2 Internal Working

An embedding model is a neural network — typically a transformer encoder — trained so that semantically similar inputs map to nearby vectors and dissimilar inputs map to distant vectors. The training objective is contrastive: given positive pairs (a question and its answer, two paraphrases) and negative pairs (unrelated content), pull the positives together and push the negatives apart in vector space.

After training, the model is frozen. To embed text, you pass it through the encoder, which outputs a fixed-size vector (typically 384, 768, 1024, 1536, 3072, or 4096 dimensions). The vector is then ready for comparison via cosine similarity or dot product.

Conceptually: imagine a giant high-dimensional sphere. Each piece of text is a point on the sphere. Similar texts cluster; unrelated texts spread out. Retrieval is finding the points nearest to your query point.

## 8.3 Why Vectors Work as Meaning

Linguistic meaning is relational. "King" is to "queen" as "man" is to "woman." That relation, in a well-trained embedding space, becomes vector arithmetic: the direction from "king" to "queen" is approximately the same as the direction from "man" to "woman." This is not a parlor trick — it reflects the fact that the model learned a geometry that mirrors human semantic structure.

For retrieval, the relational property matters less than the *similarity* property. We rarely do arithmetic on embeddings in production. We do nearest-neighbor search.

## 8.4 Similarity Metrics in Practice

**Cosine similarity** is the angle between two vectors, normalized to [-1, 1]. It is the default in most production systems because it is robust to vector magnitude and works naturally on normalized embeddings.

**Dot product** is cosine similarity multiplied by both magnitudes. If both vectors are unit-normalized, dot product equals cosine similarity, and dot product is computed faster on most hardware. Most modern embedding models output normalized vectors precisely so you can use dot product.

**Euclidean (L2) distance** is the straight-line distance. Less common for semantic search but sometimes used in image retrieval or when vector magnitude itself carries meaning.

The choice rarely affects results when embeddings are normalized. The choice does affect index format and query speed; vector databases let you pick.

## 8.5 Major Embedding Models

**OpenAI embeddings** — text-embedding-3-small and text-embedding-3-large. Pay-per-token, no infrastructure to manage, strong out-of-the-box quality. The "small" model at 1536 dimensions hits a great cost/quality point. Both support Matryoshka truncation — you can store fewer dimensions for cost savings without re-embedding.

**Cohere Embed v3** — strong multilingual support, separate models for documents and queries (an asymmetric retrieval setup that improves quality).

**Voyage AI** — leaderboard-topping embedding models with strong domain-specific variants (code, finance, law).

**BGE (BAAI General Embeddings)** — leading open-source family. bge-large, bge-m3 (multilingual + multi-vector), and reranker variants. Free to self-host. m3 is notable for handling 8K context.

**E5 (Microsoft)** — open-source family with multilingual variants. e5-mistral-7b uses a Mistral backbone for strong reasoning-heavy retrieval.

**Instructor models (HKUNLP)** — accept an instruction alongside the text ("Represent this medical question for retrieval"), letting one model serve many domains.

**Sentence-Transformers (SBERT)** — the classic open-source family, still widely used. Smaller models like all-MiniLM-L6-v2 produce 384-dim embeddings cheaply.

**Nomic, Jina, Mixedbread** — newer open-source labs producing competitive embeddings, often with permissive licenses.

## 8.6 Embedding Generation in Production

The ingestion pipeline:
1. Source documents from your data systems.
2. Chunk them (Chapter 10 covers chunking strategy in depth).
3. Embed each chunk.
4. Store embeddings plus metadata in a vector database.

Throughput considerations:
- A single A10G GPU can embed thousands of chunks per second with a small embedding model.
- Batching matters; embedding 32 chunks at once is much more than 32× faster than one at a time.
- API-based embedding is cheap until volume is huge; self-hosted wins at scale but requires GPU operations expertise.

## 8.7 Asymmetric vs Symmetric Embeddings

**Symmetric** — the same model embeds both queries and documents. Used when query and document are similar in length and style.

**Asymmetric** — separate (or differently prompted) encoders for queries and documents. Used when queries are short ("how do I reset my password") and documents are long (a passage of help docs). Asymmetric setups typically outperform symmetric for question-answering retrieval.

Practical pattern: prefix queries with a marker ("query: how do I reset my password") and documents with another ("passage: To reset your password, navigate to..."). Many BGE and E5 models expect this convention.

## 8.8 Multilingual Embeddings

Multilingual embedding models map content from different languages into a shared space. "Wie zurücksetzen mein Passwort" and "How do I reset my password" should produce nearby vectors.

Strong choices: BGE-M3, Cohere multilingual, OpenAI text-embedding-3, paraphrase-multilingual-MiniLM. Quality varies by language; English is always best, major European languages are good, lower-resource languages range from acceptable to poor.

## 8.9 Image and Multimodal Embeddings

CLIP-style models embed images and text into a shared space. A photo of a dog and the text "a dog" produce nearby vectors. This enables cross-modal retrieval: search images with text, search text with images.

Multimodal embeddings extend this to combinations: image-plus-caption, document-screenshot-plus-text. Modern multimodal RAG often uses image embeddings to retrieve scanned PDF pages directly, skipping OCR.

## 8.10 Production Architecture

```
   Source documents
        |
   Document loader (PDF, HTML, MD, code, audio, image)
        |
   Chunker (size, overlap, structure-aware)
        |
   Metadata enricher (source, timestamp, author, ACL)
        |
   Embedding model (batched, GPU-accelerated)
        |
   Vector DB (Pinecone, Qdrant, Weaviate, etc.)
        |
   Optional: BM25 index for hybrid retrieval
        |
   Optional: Knowledge graph for entity links
```

Each step is independently scalable and replaceable. Ingestion is usually batch (one-time backfill plus incremental); retrieval is online and latency-sensitive.

## 8.11 Tradeoffs

- **Dimensionality.** Higher dimensions capture more nuance but cost more storage and bandwidth. Matryoshka models let you pick at query time.
- **API vs self-hosted.** API is fast to start. Self-hosted is cheaper at scale (millions of documents) and lets you fine-tune.
- **General vs domain-specific.** Specialized models (legal, medical, code) outperform general models in their domains. Pay the cost of evaluation to find out which side you are on.
- **Asymmetric vs symmetric.** Asymmetric helps Q&A; symmetric is fine for clustering and de-duplication.

## 8.12 Scaling Challenges

- **Storage.** Ten million 1536-dim float32 vectors = 60 GB raw. Quantization (PQ, scalar quantization, binary embeddings) cuts this 4× to 32× with modest quality loss.
- **Index build time.** Building an ANN index for hundreds of millions of vectors can take hours. Plan for incremental updates rather than full rebuilds.
- **Refresh cycles.** When you update the embedding model, you must re-embed everything. Budget the cost up front.

## 8.13 Security Concerns

- Embeddings can leak information. Membership inference attacks can sometimes determine whether a given document was in your index. Mitigate with access controls and, if needed, differential privacy noise.
- Vector indices often lack the access control sophistication of databases. Enforce ACLs at the application layer, filtering vector results by user-visible scopes.
- Tenant isolation: separate indices per tenant for hard isolation, or include tenant ID in every vector's metadata for soft isolation with filtering.

## 8.14 Deployment Guide

For API-based embedding, deployment is trivial — call the endpoint. For self-hosted, package the embedding model inside an inference server (Triton, TEI, vLLM in embedding mode), expose a gRPC or REST endpoint, autoscale based on queue depth. Co-locate with the vector DB if possible to minimize network latency during ingestion.

## 8.15 Monitoring Strategy

- Embedding generation throughput and latency.
- Embedding model version and config (catch silent drift).
- Index size, query latency, recall (offline evaluation against a golden set).
- Hit rate from the application — how often does retrieval surface useful chunks.
- Cost per million embeddings.

## 8.16 Cost Optimization

- Use smaller dimensions if quality allows. Matryoshka models let you truncate.
- Use binary or scalar quantization for storage-heavy workloads.
- Batch aggressively during ingestion.
- Self-host once token volume justifies the operational overhead.
- Re-embed selectively (only changed documents) rather than full rebuilds.

## 8.17 Interview Questions

- Why are embeddings useful for search?
- Explain cosine similarity and when you would use dot product instead.
- What changes when you switch from a 384-dim model to a 1536-dim model?
- Why use asymmetric embeddings for question-answering?
- How do you handle the embedding-model upgrade migration?

## 8.18 Hands-on Exercises

1. Estimate storage cost for 50M chunks at 1024 dimensions in float32. Now repeat with int8.
2. Sketch the ingestion pipeline for 100k PDFs. Identify the throughput bottleneck.
3. Design an embedding-version-rollover plan that does not interrupt production search.

## 8.19 Common Mistakes

- Mixing embeddings from different models in the same index.
- Forgetting to normalize embeddings when the model expects normalized input.
- Re-embedding the entire corpus when only a small fraction changed.
- Skipping the asymmetric "query:" / "passage:" prefixes that the model expects.
- Using a general-purpose model for a domain-specific corpus when a specialized model exists.

## 8.20 Enterprise Best Practices

Standardize on one embedding model per index. Record the model name and version in vector metadata. Build a re-embedding pipeline before you need one. Measure retrieval quality on a real evaluation set, not vendor benchmarks. Treat the embedding model upgrade as a planned migration, not an emergency.
