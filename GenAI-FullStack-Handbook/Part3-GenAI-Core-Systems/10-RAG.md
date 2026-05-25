# Chapter 10 — Retrieval Augmented Generation (RAG)

## 10.1 Concept Explanation

Retrieval Augmented Generation is the architecture pattern where an LLM is given relevant context — documents, passages, structured data — fetched from an external store at query time, instead of relying solely on what was baked into its weights during training. RAG is the single most important pattern in production GenAI. It makes models accurate about facts they were not trained on, scalable across data that updates daily, and auditable because every claim can be traced to a source.

The slogan: "Don't fine-tune knowledge into the model; retrieve it." Fine-tuning teaches behavior; retrieval supplies facts. Conflating the two is the most common architectural mistake in early-stage GenAI projects.

## 10.2 RAG Architecture

```
                INGESTION (offline, batch)
   Documents -> Loaders -> Chunker -> Enricher -> Embedder -> Vector DB
                                                          \
                                                           -> BM25 index
                                                           -> Knowledge graph

                QUERY (online, latency-sensitive)
   User question
         |
   Optional: query rewriter / decomposer
         |
   Retriever (vector + BM25 + filters)
         |
   Reranker (cross-encoder)
         |
   Context assembler (top-k chunks + system prompt + user question)
         |
   LLM
         |
   Response with citations
         |
   Optional: validator / hallucination check
         |
   Return to user
```

Every box on this diagram is a tunable component. Production RAG is the discipline of measuring each box and improving the weakest.

## 10.3 Document Ingestion

Source documents come in PDFs, HTML, Word, Markdown, Notion, Confluence, Slack, SharePoint, Google Drive, code repositories, customer support tickets, transcripts, audio, images, scanned PDFs. Each format has its own extraction quirks.

Tools commonly used:
- **Unstructured.io** — robust multi-format parser, handles tables and layout.
- **LlamaParse / LlamaIndex parsers** — strong for PDFs.
- **PyMuPDF, pdfplumber** — programmatic PDF parsing.
- **Apache Tika** — broad format support, JVM-based.
- **OCR (Tesseract, AWS Textract, Google Document AI)** — for scanned documents.
- **Whisper** — audio transcription.

Tables, headers, footers, and footnotes are the failure points. A good ingestion pipeline preserves structure (tables as tables, lists as lists) instead of flattening everything to plain text. Modern systems often retain page numbers, headings, and bounding boxes for richer citations.

## 10.4 Chunking — The Most Underrated Lever

A chunk is a unit of retrievable content. Too small and chunks lack context; too big and the model wastes tokens on irrelevant text.

Strategies:
- **Fixed-size chunking.** Split every N tokens. Simple, fast, naive.
- **Sliding window with overlap.** N tokens per chunk with M tokens overlap. Reduces information lost at boundaries.
- **Sentence or paragraph chunking.** Respects natural boundaries.
- **Semantic chunking.** Group sentences whose embeddings are similar; split where similarity drops.
- **Structure-aware chunking.** Use document structure (headings, sections, code blocks, table rows) to define chunks.
- **Hierarchical chunking.** Store small chunks for matching, but retrieve and pass larger parent sections to the LLM ("parent-document retrieval").

Best practice for most workloads: structure-aware chunking with reasonable size (300–800 tokens) and modest overlap (10–20%). Add parent-document retrieval if accuracy demands more context.

## 10.5 Metadata Enrichment

Every chunk should carry metadata: source document ID, document title, section heading, page number, last modified date, author, ACL tags, language, content type. This metadata enables filtering, ranking signals, and citation rendering.

Generated metadata helps too: an LLM can write a one-sentence summary of each chunk that serves as an extra retrieval signal. Tags ("legal opinion," "API reference," "incident report") improve filtering precision. The cost is one-time at ingestion; the benefit is per-query.

## 10.6 Embedding and Indexing

Covered in Chapter 8 (embedding models) and Chapter 9 (vector databases). The choice of embedding model determines retrieval quality; the choice of vector DB determines operational characteristics.

## 10.7 Query Processing

The user's raw question is often suboptimal. Two improvements:

**Query rewriting.** An LLM rewrites the question to be retrieval-friendly. "Hey can you tell me how to reset my password I forgot it" becomes "password reset procedure." HyDE (Hypothetical Document Embeddings) goes further — generate a hypothetical answer and embed *that* for retrieval, on the theory that the answer is more semantically similar to the source than the question.

**Query decomposition.** Complex questions become multiple sub-queries. "Compare our pricing to Salesforce's last quarter" becomes "our pricing tiers" + "Salesforce pricing tiers Q4." Retrieve for each, merge contexts. Essential for multi-hop questions.

## 10.8 Retrieval

Vector search returns the top-k chunks by embedding similarity. BM25 returns the top-k by keyword overlap. Filters restrict to allowed documents. Combine signals via score fusion or Reciprocal Rank Fusion (RRF).

Hybrid retrieval (vector + BM25) routinely outperforms either alone, especially on queries containing identifiers, codes, or rare terms.

## 10.9 Reranking

The vector index returns approximate candidates; a reranker provides a more accurate second-pass ranking. Cross-encoders read the query and each candidate jointly and output a precise relevance score.

Popular rerankers: Cohere Rerank (managed), BGE-Reranker (open-source), Jina Reranker, mxbai-rerank. They cost more per call (cross-encoders cannot be precomputed) but you only run them on the top 50–100 candidates from the first stage, not the whole corpus.

Reranking typically lifts top-3 recall by 10–30 points. It is the single highest-ROI step in a RAG pipeline after hybrid retrieval.

## 10.10 Context Assembly

After retrieval and reranking, you have N relevant chunks. Assembling them into a prompt is an art.

- **Order matters.** LLMs attend more to the start and end of long contexts ("lost in the middle"). Put the most important chunks at the start or end.
- **Citations matter.** Number each chunk; instruct the model to cite by number.
- **Token budget matters.** Reserve room for the system prompt, the question, and the answer. Truncate or summarize aggressively if the budget is tight.
- **Diversity matters.** Retrieving five near-duplicate chunks wastes the budget. Use Maximum Marginal Relevance (MMR) to balance relevance and diversity.

## 10.11 Generation with Citations

The system prompt instructs the model to answer using only the provided context and cite which chunks it used. Failure to find an answer in the context should produce "I don't know" rather than a hallucination.

Citations are valuable for trust, debugging, and audit. They are also a partial defense against hallucination — when forced to point at evidence, models hallucinate less.

## 10.12 Graph RAG

Graph RAG augments document retrieval with a knowledge graph. Entities and relationships are extracted from documents at ingestion time and stored as a graph. At query time, the system traverses the graph to find related entities, then retrieves their associated documents.

Strength: handles multi-hop, relational questions ("who reports to the engineer who built the payments service"). Weakness: graph construction quality is the ceiling; bad extraction produces bad answers.

Microsoft's GraphRAG, Neo4j's LLM Graph Builder, and LangChain's graph utilities are the typical building blocks.

## 10.13 Multi-Hop RAG

Some questions require chained retrievals. "Find me the latest contract with Acme, then check whether their data sharing terms changed." This is two retrievals with a dependency.

Implementation: an LLM (or planner) decomposes the question, executes retrieval iteratively, feeds intermediate results back as context for the next retrieval. This is where RAG meets agentic patterns (Chapter 11).

## 10.14 Agentic RAG

Instead of a fixed retrieve-then-generate flow, an agent decides when and what to retrieve. The model itself calls a "retrieve" tool, possibly multiple times with refined queries, before producing the final answer.

Strength: dynamic, adaptive to question complexity. Weakness: higher latency, harder to debug, more expensive (more LLM calls).

A common middle ground: a fixed pipeline for simple queries, an agentic path for complex ones, routed by an initial classifier.

## 10.15 Memory Systems

Long-running assistants accumulate user-specific context. Memory systems store this and retrieve it like any other RAG corpus.

- **Episodic memory** — past conversations.
- **Semantic memory** — distilled facts about the user.
- **Procedural memory** — preferences and patterns of working.

Typical implementation: every conversation is summarized into a few sentences and embedded. At new conversation start, retrieve the top relevant memories and prepend to context. Update memories asynchronously after each session.

The pitfall is privacy. Memories are sensitive. Encrypt at rest, scope by user, allow deletion. Honor the right to be forgotten.

## 10.16 Production RAG Architecture

```
                Ingestion
   [Connectors] -> [Parsers] -> [Chunker] -> [Enricher] -> [Embedder]
                                                            |
                                                            v
                                              [Vector DB] + [BM25] + [Graph]

                Query
   [Client] -> [Query rewriter] -> [Retriever] -> [Reranker] ->
   [Context assembler] -> [LLM] -> [Validator] -> [Response w/ citations]
```

Every box has metrics. The discipline is treating each as a measurable, replaceable component.

## 10.17 RAG Evaluation

You cannot improve what you do not measure. Build an evaluation set: questions, ideal answers, and the source chunks that should be retrieved.

Metrics:
- **Retrieval metrics.** Recall@k (did we retrieve the right chunks), MRR (where in the ranking did they appear), nDCG (graded relevance).
- **Generation metrics.** Faithfulness (does the answer follow from the context), answer relevance (does it address the question), context precision (how much of retrieved context was actually used), groundedness (does every claim trace to a source).
- **End-to-end metrics.** Helpfulness, factual accuracy, user satisfaction.

LLM-as-judge automates much of this. Frameworks: RAGAS, TruLens, Phoenix, LangSmith evaluations.

## 10.18 Hallucination Mitigation

Layered defense:
1. Strong retrieval (relevant context narrows the model's guesswork).
2. Citation prompts force the model to ground claims.
3. Lower temperature reduces drift.
4. Refusal training — "if not in context, say I don't know."
5. Post-generation validators check claims against retrieved chunks.
6. LLM-as-judge or human review on a sample.

No single layer suffices. Production RAG layers them and measures the residual hallucination rate.

## 10.19 Scaling Challenges

- **Ingestion throughput** for billions of documents.
- **Query latency** under p99 — reranking adds 100–500ms.
- **Index refresh** for frequently changing corpora.
- **Cross-region replication** for global products.
- **Cost** of embeddings for huge corpora and rerankers for high QPS.

## 10.20 Security Concerns

- **ACL enforcement.** Retrieval must respect user permissions. Filter by ACL tags or use per-user indices.
- **PII handling.** Documents contain sensitive data; decide what is allowed in prompts.
- **Indirect prompt injection.** A poisoned document can instruct the model to misbehave. Treat retrieved content as untrusted.
- **Tenant isolation.** One tenant's documents must never leak into another's prompt.

## 10.21 Cost Optimization

- Cache embeddings; do not re-embed unchanged content.
- Cache full retrieved-then-generated answers when queries repeat.
- Use smaller embedding models if quality allows.
- Use cheaper LLMs for first-pass generation, escalate to frontier models only for hard cases.
- Compress chunks before sending to the LLM.

## 10.22 Interview Questions

- Why RAG over fine-tuning?
- Compare chunking strategies.
- Explain hybrid retrieval and why it outperforms pure vector.
- Walk through reranking and when it matters.
- How do you evaluate a RAG system?
- Design a multi-tenant RAG for an enterprise SaaS.

## 10.23 Hands-on Exercises

1. Sketch the full RAG pipeline for a customer support knowledge base.
2. Design the evaluation set: how many questions, what categories, how labeled.
3. Estimate ingestion cost for 1M documents averaging 5 pages each.
4. Plan a multi-tenant setup with 1,000 organizations sharing one vector DB.

## 10.24 Common Mistakes

- Chunking by fixed size without respecting structure.
- Skipping reranking; pure vector recall is mediocre.
- No evaluation set; flying blind on quality.
- Letting one tenant's documents leak into another's context.
- Using the same RAG path for trivial and complex queries; both suffer.

## 10.25 Enterprise Best Practices

Build the evaluation set before building the system. Treat retrieval, generation, and post-processing as separately observable components. Version chunks and embeddings. Audit citation accuracy. Have a documented re-ingestion process for when embedding models change. Establish per-tenant data residency requirements early.
