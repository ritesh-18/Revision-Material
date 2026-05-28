# 14 — AI / LLM / RAG (20 Questions)

Your AI work: the Clarity Finance agentic chatbot (Python, LangGraph, RAG over financial data), Claude (Sonnet) integration for anomaly reasoning, vector search, and data ingestion pipelines. These cover RAG end-to-end, embeddings, vector search, agents, prompt engineering, evaluation, and the production realities of LLM apps — with worked examples throughout.

**Sections**
- Part A — RAG fundamentals (Q1–Q7)
- Part B — Prompting, tools, agents (Q8–Q12)
- Part C — Quality, cost, production (Q13–Q20)

---

## Part A — RAG Fundamentals

### Q1. What is RAG and what problem does it solve?

**Answer:**
**RAG (Retrieval-Augmented Generation)** combines a retrieval system with an LLM. Instead of relying solely on the model's training knowledge, you retrieve relevant documents at query time and inject them into the prompt as context. The model answers grounded in those documents.

**Problems it solves:**

1. **Knowledge cutoff.** LLMs only know what they were trained on. RAG injects current/private data the model never saw.

2. **Private/proprietary data.** Your financial datasets, internal docs — the model wasn't trained on them. RAG retrieves them on demand.

3. **Hallucination reduction.** When the model has the relevant facts in context, it's far less likely to make things up. (Doesn't eliminate hallucination, but reduces it substantially.)

4. **Attribution.** You can cite which retrieved documents an answer came from — critical for finance, legal, medical.

5. **Cost vs fine-tuning.** Updating knowledge via RAG (add a document to the index) is far cheaper than retraining/fine-tuning the model.

**The core loop:**
```
   User query
       │
       ▼
   Embed query ──▶ Vector search ──▶ Top-K relevant chunks
       │                                    │
       └──────────────┬─────────────────────┘
                      ▼
   Prompt = system instructions + retrieved chunks + user query
                      │
                      ▼
                    LLM
                      │
                      ▼
   Grounded answer (+ citations)
```

**For your Clarity Finance project:** users ask natural-language questions about uploaded financial datasets. RAG retrieves the relevant rows/documents/summaries and feeds them to the LLM, which answers grounded in the actual data rather than hallucinating numbers.

**Follow-up 1: Why not just put all the data in the prompt?**
Context windows are finite (even large ones) and cost scales with tokens. A 10,000-document knowledge base won't fit, and even if it did, you'd pay for all those tokens on every query and the model's accuracy degrades with irrelevant context ("lost in the middle"). RAG retrieves only the ~5-20 most relevant chunks — cheaper, faster, and more accurate.

**Follow-up 2: When is RAG NOT the right approach?**
When the task needs the model to *learn a skill or style* rather than *access facts* — that's fine-tuning territory. When the knowledge is small and static (just put it in the system prompt). When you need reasoning over the *entire* corpus at once (RAG retrieves fragments; some analytical questions need the whole dataset — use SQL/aggregation, possibly with the LLM generating the query).

---

### Q2. Walk me through a complete RAG architecture, ingestion to answer.

**Answer:**

**Phase 1 — Ingestion (offline, when documents are added):**
```
   Documents ──▶ Loaders ──▶ Chunking ──▶ Embedding ──▶ Vector DB
   (PDF, CSV,     (extract     (split into   (embed each   (store vectors
    DB rows)       text)        passages)     chunk)        + metadata)
```
1. **Load** — extract text from sources (PDF parsers, CSV readers, DB queries).
2. **Chunk** — split into passages (Q5).
3. **Embed** — convert each chunk to a vector via an embedding model.
4. **Store** — save vectors + original text + metadata in a vector DB.

**Phase 2 — Retrieval & generation (online, per query):**
```
   Query ──▶ Embed query ──▶ Vector search (top-K) ──▶ [optional rerank] ──▶
   Build prompt (system + chunks + query) ──▶ LLM ──▶ Answer + citations
```

**Code — sketch of the full pipeline:**
```python
# Ingestion
def ingest(documents):
    for doc in documents:
        text = extract_text(doc)
        chunks = chunk_text(text, size=512, overlap=50)
        for chunk in chunks:
            vector = embedding_model.embed(chunk.text)
            vector_db.upsert(
                id=chunk.id,
                vector=vector,
                metadata={"text": chunk.text, "source": doc.id, "page": chunk.page},
            )

# Query
def answer(query: str):
    query_vec = embedding_model.embed(query)
    results = vector_db.search(query_vec, top_k=8)
    context = "\n\n".join(f"[{r.metadata['source']}] {r.metadata['text']}" for r in results)
    prompt = f"""Answer using ONLY the context below. Cite sources. If the answer isn't in the context, say so.

Context:
{context}

Question: {query}"""
    response = llm.complete(prompt)
    return response, [r.metadata["source"] for r in results]
```

**Components that make it production-grade:**
- **Metadata filtering** — restrict retrieval by tenant, date, document type (critical for your multi-tenant + RBAC).
- **Hybrid search** — combine vector + keyword (Q6).
- **Reranking** — re-score top candidates with a cross-encoder (Q7).
- **Caching** — cache embeddings and frequent query results.
- **Evaluation** — measure retrieval quality and answer faithfulness (Q14).

**Follow-up 1: Why separate ingestion (offline) from retrieval (online)?**
Embedding documents is expensive and slow; you do it once when documents are added/changed, not per query. At query time, you only embed the (short) query and do a fast vector search. This separation is what makes RAG responsive — the heavy lifting is precomputed.

**Follow-up 2: How do you handle document updates?**
Re-chunk and re-embed changed documents; upsert the new vectors; delete vectors for removed content. Track document versions and chunk IDs so you can update precisely. For frequently-changing data, this is the hard part of RAG ops — stale vectors return outdated answers. Some systems re-index on a schedule; others use change events to update incrementally.

---

### Q3. What are embeddings and how do they work?

**Answer:**
An **embedding** is a dense vector (list of floats, e.g., 768 or 1536 dimensions) that represents the *meaning* of text. Texts with similar meaning have vectors that are close together in this high-dimensional space.

**Key property: semantic similarity = vector proximity.**
- "How do I reset my password?" and "I forgot my login credentials" have different words but similar meaning → close vectors.
- "password reset" and "banana recipe" → distant vectors.

**How they're produced:**
An embedding model (a neural network — often a transformer encoder) is trained so that semantically similar texts map to nearby vectors. Models: OpenAI `text-embedding-3-small/large`, Cohere embed, open-source `sentence-transformers` (e.g., `all-MiniLM`, `bge`, `e5`).

**Measuring similarity:**
- **Cosine similarity** — the cosine of the angle between two vectors. Most common; ranges -1 to 1 (1 = identical direction). Invariant to magnitude.
- **Dot product** — similar, magnitude-sensitive.
- **Euclidean (L2) distance** — straight-line distance; smaller = more similar.

**Code — embedding and comparing:**
```python
from openai import OpenAI
import numpy as np

client = OpenAI()

def embed(text):
    return np.array(client.embeddings.create(
        model="text-embedding-3-small", input=text
    ).data[0].embedding)

def cosine(a, b):
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))

q = embed("how do I reset my password")
d1 = embed("forgot my login credentials")
d2 = embed("best banana bread recipe")
print(cosine(q, d1))   # ~0.7 (similar)
print(cosine(q, d2))   # ~0.1 (unrelated)
```

**Why high dimensions?**
More dimensions can encode more nuance — different "directions" capture topic, sentiment, formality, entities, etc. Trade-off: more dimensions = more storage and slower search. Common sizes: 384 (small/fast), 768, 1536 (OpenAI), 3072 (large).

**Follow-up 1: Can you compare embeddings from different models?**
No. Each model has its own vector space; a vector from OpenAI's model means nothing to Cohere's. You must embed both query and documents with the *same* model. If you switch embedding models, you must re-embed your entire corpus.

**Follow-up 2: What's the difference between embedding models for retrieval vs general-purpose?**
Retrieval-optimized models (e5, bge, OpenAI's retrieval embeddings) are trained on query-document pairs — they're tuned so a *question* embeds close to its *answer passage* (which use different words). Some are "asymmetric" (different encoding for queries vs documents). General sentence embeddings optimize for sentence similarity, which isn't quite the same task. Use retrieval-tuned models for RAG.

---

### Q4. What's a vector database and how does vector search actually work?

**Answer:**
A **vector database** stores embeddings and supports fast **approximate nearest neighbor (ANN)** search — "find the K vectors most similar to this query vector."

**Why not just compute cosine against every vector?**
Exact search (compare query to every stored vector) is O(N) — fine for thousands, too slow for millions. ANN algorithms trade a tiny bit of accuracy for massive speed.

**ANN algorithms:**
- **HNSW (Hierarchical Navigable Small World)** — a layered graph; search navigates from coarse to fine. Fast, high recall, memory-heavy. The most popular.
- **IVF (Inverted File Index)** — cluster vectors; search only the nearest clusters. Tunable speed/accuracy.
- **PQ (Product Quantization)** — compress vectors to save memory, with some accuracy loss. Often combined with IVF.

**Vector DB options:**
- **Dedicated:** Pinecone (managed), Weaviate, Qdrant, Milvus, Chroma (lightweight/local).
- **Postgres + pgvector** — vector search inside Postgres. Great if you already use Postgres (your stack!) — keeps vectors alongside relational data, supports metadata filtering with SQL.
- **Elasticsearch / OpenSearch** — vector + keyword in one engine (good for hybrid search).
- **Redis** — vector search module.

**Code — pgvector (fits your Postgres-heavy stack):**
```sql
CREATE EXTENSION vector;

CREATE TABLE documents (
  id        uuid PRIMARY KEY,
  tenant_id uuid NOT NULL,
  content   text NOT NULL,
  embedding vector(1536)        -- OpenAI dimension
);

-- HNSW index for fast ANN
CREATE INDEX ON documents USING hnsw (embedding vector_cosine_ops);

-- Search: nearest neighbors with metadata filter (tenant isolation!)
SELECT content, 1 - (embedding <=> $1) AS similarity
FROM documents
WHERE tenant_id = $2                  -- pre-filter by tenant
ORDER BY embedding <=> $1             -- <=> is cosine distance
LIMIT 8;
```

The `<=>` operator is cosine distance; `<->` is L2; `<#>` is negative inner product.

**Follow-up 1: Why is pgvector attractive for your stack specifically?**
You already run Postgres with multi-tenant schema isolation. pgvector lets you store embeddings next to relational data, filter by `tenant_id` in the same query (combining metadata filtering with vector search), and avoid running a separate vector DB. For moderate scale (millions of vectors), pgvector with HNSW is plenty. You only need a dedicated vector DB at very large scale or for advanced features.

**Follow-up 2: What's the recall/speed trade-off in ANN?**
ANN finds *approximate* nearest neighbors — it might miss a few true top-K results. Tuning parameters (HNSW's `ef_search`, IVF's `nprobe`) trades recall for speed: higher values = more accurate but slower. For RAG, 95-99% recall is usually fine — missing one of the top-8 chunks rarely changes the answer. Measure recall against exact search on a sample to validate your settings.

---

### Q5. What are chunking strategies and why do they matter so much?

**Answer:**
**Chunking** = splitting documents into passages before embedding. It's one of the highest-leverage decisions in RAG — bad chunking ruins retrieval no matter how good your model is.

**Why chunk at all?**
- Embedding models have input limits.
- Whole documents are too coarse — a 50-page PDF embedded as one vector loses specificity.
- You want to retrieve the *relevant passage*, not the whole document.

**Strategies:**

1. **Fixed-size** — split every N tokens/characters. Simple but cuts mid-sentence, mid-idea.

2. **Fixed-size with overlap** — N tokens with M-token overlap between chunks. Overlap prevents losing context at boundaries. The common default (e.g., 512 tokens, 50 overlap).

3. **Sentence/paragraph-based** — split on natural boundaries. Preserves semantic units.

4. **Recursive** — split on a hierarchy of separators (paragraphs → sentences → words) until chunks fit the size. LangChain's `RecursiveCharacterTextSplitter`.

5. **Semantic chunking** — split where the topic shifts (detected by embedding similarity between sentences). More expensive, often better.

6. **Structure-aware** — for structured docs (markdown headers, code functions, CSV rows), chunk along the structure. For your financial data, chunk by record/section/table.

**Trade-offs:**
- **Too small** — chunks lack context; retrieval gets fragments that don't fully answer.
- **Too large** — chunks contain irrelevant text, diluting the embedding's specificity and wasting context tokens.

**Code — recursive chunking with overlap:**
```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=512,
    chunk_overlap=50,
    separators=["\n\n", "\n", ". ", " ", ""],   # try paragraph, then sentence, then word
)
chunks = splitter.split_text(document_text)
```

**For your financial data:** structure-aware chunking — keep a transaction record or a financial statement section together, attach metadata (date, account, document type) for filtering. Splitting a balance sheet mid-row would be useless.

**Follow-up 1: How do you choose chunk size?**
Match it to the granularity of your answers and your embedding model's sweet spot. For Q&A over prose, 256-512 tokens is common. For dense reference data, smaller. Experiment: build an eval set, try sizes, measure retrieval quality. There's no universal best — it depends on document structure and query types. Overlap of ~10-20% of chunk size is a reasonable default.

**Follow-up 2: What's the "parent-child" or "small-to-big" chunking trick?**
Embed small chunks (precise retrieval) but return the larger parent chunk (more context for the LLM). You retrieve on a focused 128-token chunk but feed the model the surrounding 512-token paragraph. Best of both: precise matching + rich context. LangChain's `ParentDocumentRetriever` implements this.

---

### Q6. Semantic vs keyword vs hybrid search — when use each?

**Answer:**

**Keyword search (BM25, full-text):**
- Matches exact terms and their statistical importance.
- Great for: exact terms, names, IDs, codes, jargon, acronyms.
- Fails on: synonyms, paraphrases ("car" won't match "automobile").

**Semantic (vector) search:**
- Matches meaning via embeddings.
- Great for: paraphrases, conceptual queries, natural language.
- Fails on: exact identifiers (a product code embeds to something generic), rare jargon the embedding model doesn't understand well.

**Hybrid search:**
- Run both; combine scores (e.g., Reciprocal Rank Fusion).
- Best of both: semantic understanding + exact-term precision.
- The production default for serious RAG.

**Worked example of why hybrid wins:**
Query: "What was the Q3 2025 EBITDA for product SKU-4471?"
- Semantic search understands "EBITDA", "Q3 2025" conceptually but might miss the exact SKU.
- Keyword search nails "SKU-4471" exactly but doesn't understand "EBITDA" relationships.
- Hybrid retrieves documents matching both the concept and the exact identifier.

**Code — Reciprocal Rank Fusion (RRF):**
```python
def reciprocal_rank_fusion(keyword_results, vector_results, k=60):
    scores = {}
    for rank, doc in enumerate(keyword_results):
        scores[doc.id] = scores.get(doc.id, 0) + 1 / (k + rank)
    for rank, doc in enumerate(vector_results):
        scores[doc.id] = scores.get(doc.id, 0) + 1 / (k + rank)
    return sorted(scores.items(), key=lambda x: x[1], reverse=True)
```

**Implementation options:**
- Elasticsearch/OpenSearch — both keyword (BM25) and vector in one engine.
- pgvector + Postgres full-text search — combine in SQL.
- Weaviate, Qdrant — built-in hybrid.

**Follow-up 1: Why is keyword search still relevant in the LLM era?**
Because embeddings are bad at exact matches. Identifiers, error codes, proper nouns, rare terms — these need exact matching. For your financial data full of account numbers, SKUs, and precise figures, keyword search catches what semantic search blurs. The "vectors solve everything" assumption fails on precision-critical retrieval.

**Follow-up 2: How do you combine scores from two different ranking systems?**
The scores aren't directly comparable (BM25 scores vs cosine similarities have different scales). Reciprocal Rank Fusion solves this by using *rank position* rather than raw scores — robust and parameter-light. Alternatives: normalize both score distributions then weight-sum, or train a learned combiner. RRF is the pragmatic default.

---

### Q7. What is reranking and when do you need it?

**Answer:**
**Reranking** is a second-stage scoring step. First-stage retrieval (vector/keyword) fetches a larger candidate set (say top-50) quickly; a **reranker** then re-scores those candidates more accurately and keeps the best (say top-8).

**Why two stages?**
- First-stage retrieval is fast but coarse. Bi-encoders (embeddings) encode query and document *separately* — they can't model fine-grained interactions.
- A **cross-encoder** reranker takes the query AND a document *together* and scores their relevance directly. Much more accurate, but too slow to run against the whole corpus.
- So: fast retrieval narrows to 50 candidates; slow-but-accurate reranker picks the best 8.

```
   Query ──▶ Vector search (top 50, fast) ──▶ Cross-encoder rerank (top 8, accurate) ──▶ LLM
```

**Bi-encoder vs cross-encoder:**
- **Bi-encoder** — `embed(query)` and `embed(doc)` separately, compare. Fast (docs pre-embedded), less accurate.
- **Cross-encoder** — `score(query, doc)` jointly. Slow (must run per pair at query time), more accurate.

**Rerankers:**
- Cohere Rerank (managed API).
- Open-source cross-encoders (`bge-reranker`, `ms-marco-MiniLM`).
- LLM-based reranking (ask the LLM to rate relevance — expensive but flexible).

**Code:**
```python
import cohere
co = cohere.Client(api_key)

def retrieve_and_rerank(query, vector_db):
    candidates = vector_db.search(embed(query), top_k=50)   # fast first stage
    reranked = co.rerank(
        query=query,
        documents=[c.text for c in candidates],
        top_n=8,
        model="rerank-english-v3.0",
    )
    return [candidates[r.index] for r in reranked.results]   # best 8
```

**Follow-up 1: When is reranking worth the added latency/cost?**
When retrieval quality directly impacts answer quality and you're seeing relevant docs ranked too low (in positions 10-30, not top-8). Reranking adds ~100-300ms and per-document cost. For high-stakes Q&A (finance, legal) where getting the right context matters, it's worth it. For casual chat, maybe not. Measure: does reranking improve your retrieval eval metrics enough to justify the latency?

**Follow-up 2: Can the LLM itself do reranking?**
Yes — prompt it to score or order candidates by relevance. Pros: flexible, can use reasoning. Cons: expensive (more LLM calls), slower, and you're using a generation model for a ranking task it's not optimized for. Dedicated cross-encoder rerankers are usually a better speed/cost/quality trade-off. LLM reranking shines when relevance requires complex reasoning the reranker can't do.

---

## Part B — Prompting, Tools, Agents

### Q8. What are the fundamentals of prompt engineering?

**Answer:**
Prompt engineering is structuring inputs to get reliable, high-quality outputs from an LLM.

**Core techniques:**

1. **Clear instructions** — be specific about the task, format, and constraints. "Summarize in 3 bullet points" beats "summarize."

2. **Role/system prompt** — set the model's persona and rules. "You are a financial analyst. Answer only from the provided data. If unsure, say so."

3. **Few-shot examples** — show input/output examples. The model pattern-matches. Powerful for format consistency and tricky tasks.

4. **Structured output** — request JSON/XML; use schema enforcement or tool-calling for guaranteed structure.

5. **Chain-of-thought (CoT)** — "Think step by step before answering." For reasoning tasks, asking the model to reason first improves accuracy.

6. **Delimiters** — separate instructions from data with clear markers (XML tags, triple quotes) to prevent confusion and injection.

7. **Constraints and grounding** — "Use ONLY the context. If the answer isn't there, say 'I don't know.'" Reduces hallucination.

**Code — a well-structured RAG prompt (Claude-style with XML):**
```python
prompt = f"""You are a financial analysis assistant. Answer the user's question using ONLY the data in <context>. 

Rules:
- Cite the source document for each fact: [source: doc_id].
- If the answer is not in the context, say "I don't have that information."
- Show numbers exactly as they appear; do not estimate.

<context>
{retrieved_chunks}
</context>

<question>
{user_question}
</question>

Think through the relevant data, then give a concise answer with citations."""
```

**Anti-patterns:**
- Vague instructions → inconsistent output.
- Mixing instructions and data without delimiters → prompt injection risk.
- Over-stuffing the prompt → "lost in the middle" (model ignores middle context).
- Asking for reasoning AND strict format simultaneously without structure → conflicts.

**Follow-up 1: Why does chain-of-thought improve accuracy?**
LLMs generate token by token; complex answers benefit from "working space." By generating reasoning steps first, the model conditions its final answer on its own intermediate reasoning — like showing work in math. For multi-step problems (your agentic financial reasoning), CoT substantially improves correctness. Newer "reasoning models" do this internally.

**Follow-up 2: How do you prevent prompt injection in a RAG system?**
Retrieved documents (or user input) might contain "ignore previous instructions and..." Defenses: (1) clear delimiters separating instructions from data (XML tags); (2) instruction in the system prompt that data is data, not commands; (3) treat the LLM's output as untrusted (validate, sandbox tool calls); (4) don't give the LLM dangerous capabilities without guardrails. No defense is perfect; assume injection is possible and limit blast radius.

---

### Q9. How does LLM function calling / tool use work?

**Answer:**
**Function calling** (tool use) lets an LLM invoke external functions. You define tools with schemas; the model decides when to call them and produces structured arguments; your code executes the function and returns the result; the model uses the result to answer.

**The flow:**
```
   1. You send: user message + tool definitions
   2. Model responds: "call get_stock_price(symbol='AAPL')"  (structured)
   3. Your code: execute get_stock_price('AAPL') → 182.50
   4. You send: the result back to the model
   5. Model responds: "AAPL is trading at $182.50"
```

The model doesn't execute anything — it outputs *which* tool to call and *what arguments*. Your code does the execution. This keeps control and safety in your hands.

**Code — tool use with Claude:**
```python
import anthropic
client = anthropic.Anthropic()

tools = [{
    "name": "query_financial_data",
    "description": "Query the user's financial dataset by metric and period",
    "input_schema": {
        "type": "object",
        "properties": {
            "metric": {"type": "string", "enum": ["revenue", "ebitda", "expenses"]},
            "period": {"type": "string", "description": "e.g. Q3-2025"},
        },
        "required": ["metric", "period"],
    },
}]

response = client.messages.create(
    model="claude-opus-4-7",
    max_tokens=1024,
    tools=tools,
    messages=[{"role": "user", "content": "What was Q3 2025 revenue?"}],
)

# Model returns a tool_use block
for block in response.content:
    if block.type == "tool_use":
        result = query_financial_data(**block.input)   # YOUR code runs it
        # Send result back
        follow_up = client.messages.create(
            model="claude-opus-4-7",
            max_tokens=1024,
            tools=tools,
            messages=[
                {"role": "user", "content": "What was Q3 2025 revenue?"},
                {"role": "assistant", "content": response.content},
                {"role": "user", "content": [{
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": str(result),
                }]},
            ],
        )
```

**This is the foundation of agents** — the model uses tools in a loop to accomplish tasks. Your Clarity Finance bot uses tools for SQL queries, chart generation, document retrieval.

**Follow-up 1: How does the model "know" when to call a tool?**
The model is trained to recognize when a query needs external data/action and to emit a tool-call in the expected format. Your tool *descriptions* are crucial — clear descriptions help the model choose the right tool with the right arguments. Vague descriptions = wrong or missing tool calls. Treat tool descriptions as prompt engineering.

**Follow-up 2: What's the security concern with tool use?**
The model decides which tools to call with what arguments — if a tool can do something dangerous (delete data, spend money, run arbitrary SQL), a manipulated model (prompt injection) could misuse it. Defenses: least-privilege tools (read-only where possible), validate/sanitize arguments, require confirmation for destructive actions, sandbox execution, and never expose raw SQL execution without guardrails (parameterize, allowlist).

---

### Q10. What is an AI agent and how does it differ from a single LLM call?

**Answer:**
An **agent** is an LLM that operates in a **loop**, using tools and its own previous outputs to accomplish a multi-step task — rather than a single prompt-response.

**Single LLM call:** input → output. One shot.

**Agent loop:**
```
   ┌─────────────────────────────────────────┐
   │  1. LLM observes the goal + current state│
   │  2. LLM decides next action (tool call)  │
   │  3. Execute the tool                      │
   │  4. Feed the result back to the LLM       │
   │  5. Repeat until the LLM decides it's done│
   └─────────────────────────────────────────┘
```

**Key capabilities:**
- **Planning** — break a goal into steps.
- **Tool use** — call functions, APIs, search, code execution.
- **Memory** — track what it's done, intermediate results.
- **Reflection** — evaluate its own progress, correct course.

**Example — your Clarity Finance agent:**
```
User: "Compare Q3 revenue across our top 3 products and chart it."

Agent loop:
  → call query_data(metric=revenue, period=Q3, group_by=product)   [tool]
  → observe: [{product: A, rev: 100}, {B: 80}, {C: 60}]
  → call identify_top_n(data, n=3)                                  [tool]
  → call generate_chart(type=bar, data=top3)                        [tool]
  → observe: chart_url
  → respond: "Here's the Q3 revenue comparison [chart]. Product A led at $100K..."
```

**Patterns:**
- **ReAct (Reason + Act)** — model alternates reasoning and tool actions.
- **Plan-and-execute** — model makes a full plan first, then executes steps.
- **Multi-agent** — specialized agents (researcher, writer, critic) collaborate.

**Follow-up 1: What's the biggest challenge with agents in production?**
Reliability and cost. Each loop iteration is an LLM call — agents can run many iterations, multiplying cost and latency. They can get stuck in loops, make wrong decisions that compound, or call tools incorrectly. Mitigations: step limits, timeouts, validation between steps, human-in-the-loop for high-stakes actions, and extensive testing. Agents are powerful but harder to make reliable than single calls.

**Follow-up 2: When do you NOT need an agent?**
When the task is a single, well-defined step — just use a direct LLM call or a fixed pipeline. Agents add overhead and unpredictability. If you can express the task as "retrieve context, then answer" (RAG), you don't need the agentic loop. Reserve agents for genuinely multi-step, dynamic tasks where the steps aren't known in advance. Many "agent" use cases are better served by a fixed workflow with a couple of LLM calls.

---

### Q11. How does LangGraph work and why use it over plain LangChain?

**Answer:**
**LangGraph** models agent/LLM workflows as a **graph** (state machine): nodes are steps (LLM calls, tool executions, logic), edges define transitions (including conditional branching and loops). State flows through the graph and is updated at each node.

**Why a graph?**
- Agentic workflows are inherently stateful and cyclic (loops, retries, branches). Linear chains (basic LangChain) struggle to express "if the model wants a tool, loop back; else finish."
- Graphs make the control flow **explicit and inspectable** — you can see and debug the workflow structure.
- Supports persistence (checkpoint state between steps), human-in-the-loop (pause for approval), and streaming.

**Core concepts:**
- **State** — a shared object passed between nodes (messages, intermediate results).
- **Nodes** — functions that read and update state.
- **Edges** — transitions; can be conditional (route based on state).
- **Checkpoints** — persist state so a workflow can pause/resume.

**Code — a simple agent graph:**
```python
from langgraph.graph import StateGraph, END
from typing import TypedDict, Annotated
import operator

class AgentState(TypedDict):
    messages: Annotated[list, operator.add]

def call_model(state):
    response = llm.invoke(state["messages"])
    return {"messages": [response]}

def call_tools(state):
    last = state["messages"][-1]
    results = [execute_tool(tc) for tc in last.tool_calls]
    return {"messages": results}

def should_continue(state):
    last = state["messages"][-1]
    return "tools" if last.tool_calls else END   # loop or finish

graph = StateGraph(AgentState)
graph.add_node("model", call_model)
graph.add_node("tools", call_tools)
graph.set_entry_point("model")
graph.add_conditional_edges("model", should_continue)   # branch
graph.add_edge("tools", "model")                        # loop back
app = graph.compile()

result = app.invoke({"messages": [HumanMessage("Compare Q3 revenue and chart it")]})
```

**For your Clarity Finance project:** LangGraph orchestrates the multi-step reasoning — retrieve data, decide whether more data is needed, call chart tools, synthesize. The graph makes the "tool orchestration and intelligent decision-making" explicit and debuggable.

**Follow-up 1: Why LangGraph over hand-rolling the agent loop?**
You *can* hand-roll a `while` loop. LangGraph adds: built-in state management, checkpointing (resume long-running workflows), streaming of intermediate steps, human-in-the-loop interrupts, and a visualizable graph structure. For simple loops, hand-rolling is fine; for complex multi-step agents with branching, persistence, and observability needs, LangGraph's structure pays off. Trade-off: a framework dependency and learning curve.

**Follow-up 2: How does LangGraph handle human-in-the-loop?**
Via interrupts and checkpoints. You can configure a node to pause execution, persist the state (checkpoint), and wait for human approval before continuing. Useful for high-stakes actions: "the agent wants to execute this transaction — approve?" The graph resumes from the checkpoint once approved. This is hard to do cleanly with a plain loop.

---

### Q12. How do you mitigate hallucinations in LLM applications?

**Answer:**
**Hallucination** = the model generates plausible-sounding but false information. It's inherent to how LLMs work (they predict likely tokens, not verified facts).

**Mitigation strategies:**

1. **Grounding via RAG** — give the model the actual facts in context. "Answer from this data" hallucinates far less than "answer from memory."

2. **Explicit "I don't know" permission** — instruct: "If the answer isn't in the context, say you don't know." Without this, models fill gaps by guessing.

3. **Citations** — require the model to cite which source each claim comes from. Makes hallucinations visible (a fabricated claim has no real citation) and verifiable.

4. **Constrain to retrieved context** — "Use ONLY the provided context." Reduces the model drawing on (possibly wrong) parametric knowledge.

5. **Lower temperature** — for factual tasks, low temperature (0-0.3) makes output more deterministic and less "creative" (creativity = hallucination for facts).

6. **Verification step** — a second LLM call (or rules) checks whether the answer is supported by the context. "Faithfulness checking."

7. **Structured output for facts** — extract specific values into structured fields you can validate against the source, rather than free-form prose.

8. **Human review** for high-stakes outputs (finance, medical, legal).

**Code — faithfulness check:**
```python
def check_faithfulness(answer, context):
    check = llm.complete(f"""Is every claim in the ANSWER supported by the CONTEXT?
Respond with JSON: {{"faithful": bool, "unsupported_claims": [...]}}

CONTEXT: {context}
ANSWER: {answer}""")
    result = json.loads(check)
    if not result["faithful"]:
        log.warning("unfaithful answer", claims=result["unsupported_claims"])
    return result
```

**For finance specifically (your domain):** never let the model compute or estimate numbers freely — retrieve exact figures, have it cite them, and consider computing aggregations in code (SQL) rather than asking the LLM to do math, which it does unreliably.

**Follow-up 1: Why does giving the model facts not fully eliminate hallucination?**
The model can still misread context, combine facts incorrectly, over-generalize, or fall back on training knowledge that contradicts the context. RAG dramatically reduces but doesn't eliminate hallucination. Layering — grounding + citations + faithfulness checks + structured extraction — drives it down further, but "zero hallucination" isn't achievable with current models. Design for verification, not perfection.

**Follow-up 2: How do you handle numerical accuracy specifically?**
LLMs are unreliable at arithmetic. Best practice: don't ask the model to compute. Instead, (1) retrieve exact numbers and have the model quote them verbatim; (2) for calculations, have the model generate code/SQL that you execute (the model writes `SUM(revenue)`, your DB computes it); (3) validate any number in the output against the source. For your financial app, code-generation-then-execute is the reliable path — the model's job is understanding the question and formatting the answer, not doing math.

---

## Part C — Quality, Cost, Production

### Q13. How do you manage the context window?

**Answer:**
The **context window** is the maximum tokens the model can process at once (prompt + response). Even large windows (200K+) have limits and costs.

**Challenges:**
- **Cost** — you pay per token; large contexts are expensive per call.
- **Latency** — more tokens = slower processing.
- **"Lost in the middle"** — models attend best to the start and end of context; information buried in the middle is often ignored. Empirically demonstrated across models.
- **Hard limits** — exceed the window and the request fails.

**Strategies:**

1. **Retrieve only what's relevant** (RAG) — don't dump everything; retrieve top-K chunks.

2. **Rank by relevance, place strategically** — put the most relevant context at the start AND end (combat lost-in-the-middle), less critical in the middle.

3. **Summarize/compress** — for long conversations, summarize older turns instead of keeping full history.

4. **Token budgeting** — allocate explicitly: X tokens for system prompt, Y for retrieved context, Z for conversation history, leave room for the response.

5. **Truncate intelligently** — when over budget, drop least-relevant context, not arbitrary tail.

**Code — token budgeting for a RAG chat:**
```python
def build_prompt(system, history, retrieved, query, max_tokens=180_000, response_budget=4000):
    budget = max_tokens - response_budget
    system_tokens = count_tokens(system)
    query_tokens = count_tokens(query)
    available = budget - system_tokens - query_tokens

    # Allocate: 60% to retrieved context, 40% to history
    context = fit_to_budget(retrieved, int(available * 0.6))
    history = fit_to_budget(history, int(available * 0.4))   # keep most recent
    return assemble(system, history, context, query)
```

**Follow-up 1: What's "lost in the middle" and how do you work around it?**
Research shows LLMs have a U-shaped attention pattern — they recall information at the beginning and end of context well, but degrade for the middle. Workarounds: put the most important context first and last; reduce total context (fewer, more relevant chunks via reranking); for critical facts, repeat or emphasize them. Don't assume the model "reads everything equally."

**Follow-up 2: Should you always use the largest context window available?**
No. Larger context costs more, is slower, and risks lost-in-the-middle. Use the smallest context that contains the relevant information. RAG + reranking to get 5-10 great chunks beats stuffing 100 mediocre ones into a huge context. Large windows are useful for genuinely large single documents, not as an excuse to skip retrieval quality.

---

### Q14. How do you evaluate a RAG system?

**Answer:**
RAG has two stages to evaluate separately: **retrieval** (did we fetch the right context?) and **generation** (did we produce a good answer from it?).

**Retrieval metrics:**
- **Recall@K** — of the relevant documents, how many appear in the top-K? (Did we retrieve what we needed?)
- **Precision@K** — of the top-K retrieved, how many are relevant? (How much noise?)
- **MRR (Mean Reciprocal Rank)** — how high up is the first relevant result?
- **NDCG** — ranking quality weighted by position.

**Generation metrics:**
- **Faithfulness/groundedness** — is the answer supported by the retrieved context? (No hallucination.)
- **Answer relevance** — does the answer address the question?
- **Context relevance** — was the retrieved context relevant to the question?
- **Correctness** — is the answer factually right (vs a ground-truth answer)?

**Evaluation approaches:**
- **Golden dataset** — curated question + ideal answer + relevant docs. Run the system, compare.
- **LLM-as-judge** — use a strong model to score answers on faithfulness, relevance. Cheaper than human eval, decent correlation.
- **Human eval** — gold standard, expensive. Sample for high-stakes systems.
- **Frameworks** — RAGAS, TruLens, DeepEval automate these metrics.

**Code — LLM-as-judge faithfulness:**
```python
def evaluate_answer(question, context, answer, ground_truth=None):
    judge_prompt = f"""Rate this RAG answer on a 1-5 scale for each dimension. Respond in JSON.

Question: {question}
Retrieved context: {context}
Answer: {answer}

Dimensions:
- faithfulness: is every claim supported by the context?
- relevance: does it answer the question?
- completeness: does it cover what the context supports?"""
    return json.loads(strong_llm.complete(judge_prompt))
```

**Why eval matters:** RAG has many tunable knobs (chunk size, K, embedding model, reranker, prompt). Without measurement, you're guessing. Build an eval set early, measure every change. "Improved read query performance by 20-30%" requires measurement; same for "improved retrieval quality."

**Follow-up 1: How do you build a golden eval dataset without a lot of manual work?**
Bootstrap with LLM generation: feed documents to a strong model and ask it to generate question-answer pairs grounded in them. Review a sample for quality. This gives you a starting eval set quickly. Augment with real user queries (logged from production) over time — those are the most representative. Iterate.

**Follow-up 2: How reliable is LLM-as-judge?**
Reasonably, for relative comparisons (is config A better than B?), less so for absolute scores. Biases: prefers longer answers, its own style, position effects. Mitigations: use a strong model as judge, give clear rubrics, use pairwise comparison instead of absolute scoring, calibrate against human labels on a sample. It's a pragmatic tool for iteration speed, not a substitute for human eval on critical systems.

---

### Q15. How do you optimize LLM application cost and latency?

**Answer:**

**Cost levers:**

1. **Right-size the model.** Use the cheapest model that meets quality needs. Haiku for simple tasks, Opus for complex reasoning. Route by complexity — cheap model first, escalate only if needed.

2. **Prompt caching** — cache the static prefix of prompts (system prompt, retrieved context, examples). Subsequent calls reuse the cached prefix at a fraction of the cost. (Q19, huge for RAG.)

3. **Reduce tokens** — concise prompts, retrieve fewer/better chunks (reranking), summarize history. You pay per token both ways.

4. **Batch** — for offline workloads, batch APIs offer ~50% discounts.

5. **Cache responses** — identical/similar queries return cached answers. Semantic caching (embed the query, return cached answer if a near-identical query was seen).

**Latency levers:**

1. **Streaming** — stream tokens to the user as they generate (Q16). Perceived latency drops dramatically even if total time is the same.

2. **Smaller/faster models** where quality allows.

3. **Parallelize** — independent retrievals, tool calls run concurrently.

4. **Prompt caching** also cuts latency (cached prefix isn't reprocessed).

5. **Reduce agent loops** — each loop is a round-trip; minimize iterations.

6. **Reduce output tokens** — generation is the slow part; ask for concise answers, use `max_tokens`.

**Code — model routing by complexity:**
```python
def route_query(query):
    complexity = classify_complexity(query)   # cheap model or heuristic
    if complexity == "simple":
        return haiku.complete(query)
    elif complexity == "complex":
        return opus.complete(query)
```

**Follow-up 1: What's semantic caching and when does it help?**
Cache answers keyed by query embedding. On a new query, embed it; if a previous query is very similar (cosine > threshold), return the cached answer. Helps for FAQ-style workloads with repeated/paraphrased questions. Risk: returning a cached answer for a subtly different question. Use a high similarity threshold and scope caches carefully (per-tenant, per-context).

**Follow-up 2: How do you decide between a bigger model vs better RAG?**
Often better RAG beats a bigger model AND costs less. If the model has the right context, even a smaller model answers well. Spend effort on retrieval quality (chunking, hybrid search, reranking) before reaching for the most expensive model. Measure: does a cheaper model with great context match an expensive model with mediocre context? Usually yes, at a fraction of the cost.

---

### Q16. How do you stream LLM responses, and why does it matter?

**Answer:**
**Streaming** sends tokens to the client as the model generates them, rather than waiting for the complete response.

**Why it matters:**
- **Perceived latency.** A 10-second full response feels slow; streaming the first tokens in 0.5s and continuing feels responsive. Same total time, vastly better UX.
- **Early feedback.** Users see the answer forming; they can stop if it's going wrong.
- **Long responses.** For a multi-paragraph answer, waiting for the whole thing is painful.

**How it works:**
- The LLM API supports Server-Sent Events (SSE) or chunked streaming.
- Your backend receives tokens incrementally and forwards them to the client.
- The client renders tokens as they arrive.

**Code — streaming from Claude through a NestJS/Express endpoint:**
```python
# Python backend streaming
def stream_answer(query):
    with client.messages.stream(
        model="claude-opus-4-7",
        max_tokens=2000,
        messages=[{"role": "user", "content": query}],
    ) as stream:
        for text in stream.text_stream:
            yield text   # forward to client via SSE
```

```js
// Node/NestJS SSE endpoint
@Sse('chat')
chat(@Query('q') query: string): Observable<MessageEvent> {
  return new Observable(subscriber => {
    llmStream(query, (token) => subscriber.next({ data: token }));
  });
}
```

```js
// Client
const es = new EventSource('/chat?q=...');
es.onmessage = (e) => { output.textContent += e.data; };
```

**Considerations:**
- **Error handling mid-stream** — what if the model errors after 50 tokens? Send an error event; the client must handle partial responses.
- **Buffering at proxies** — Nginx buffers by default; set `proxy_buffering off` for SSE (section 09 Q8).
- **Tool use + streaming** — more complex; you stream text but pause for tool calls.

**Follow-up 1: SSE vs WebSocket for streaming LLM responses?**
SSE (Server-Sent Events) is simpler and sufficient for one-way streaming (server → client), which is what LLM token streaming is. WebSocket is bidirectional — overkill unless you need the client to send messages mid-stream. For a chat UI where the user sends a query and receives a streamed response, SSE is the cleaner choice. WebSocket if you need full duplex (e.g., interrupt/steer mid-generation).

**Follow-up 2: How do you handle streaming with structured output / tool calls?**
Trickier. With tool use, the model might stream text, then emit a tool call (which you must execute before continuing), then stream more. You stream the text portions but pause at tool boundaries. For pure structured JSON output, streaming partial JSON is awkward — clients can't parse incomplete JSON. Often you stream a "thinking" text portion for UX, then deliver the final structured result non-streamed.

---

### Q17. How do you add guardrails and safety to an LLM application?

**Answer:**
Guardrails constrain what goes into and comes out of the LLM.

**Input guardrails:**
- **Prompt injection detection** — flag inputs trying to override instructions ("ignore previous instructions").
- **PII detection** — catch/redact sensitive data before sending to the model.
- **Topic/scope filtering** — reject off-topic or disallowed requests ("this assistant only handles finance questions").
- **Rate limiting** — per-user/tenant caps (LLM calls are expensive).

**Output guardrails:**
- **Faithfulness/grounding check** — is the answer supported by context? (Q12)
- **PII/sensitive data filtering** — don't leak data the user shouldn't see.
- **Format validation** — enforce expected structure (JSON schema).
- **Content moderation** — flag harmful content.
- **Citation verification** — do cited sources actually support the claims?

**Tool-use guardrails:**
- **Least privilege** — tools can only do what's necessary (read-only where possible).
- **Argument validation** — sanitize/validate tool arguments before execution.
- **Human approval** for high-stakes actions.
- **Sandboxing** — execute generated code in isolated environments.

**Architecture:**
```
   Input ──▶ [Input guardrails] ──▶ LLM ──▶ [Output guardrails] ──▶ Response
                                      │
                                  [Tool guardrails]
```

**Code — input/output guardrail wrapper:**
```python
def safe_complete(query, context, tenant_id):
    # Input guardrails
    if detect_injection(query):
        return "I can't process that request."
    if not is_in_scope(query):
        return "I only answer questions about your financial data."

    answer = llm.complete(build_prompt(context, query))

    # Output guardrails
    if not check_faithfulness(answer, context):
        return "I don't have enough information to answer that confidently."
    answer = redact_pii(answer, allowed_for=tenant_id)
    return answer
```

**Follow-up 1: How do you handle the model being asked to do something outside its scope?**
Scope enforcement in the system prompt ("you only answer finance questions") plus an input classifier that rejects off-topic queries before they hit the expensive model. Defense in depth: even if the prompt instruction is bypassed (injection), the classifier catches it. For your finance bot, this prevents misuse and keeps costs down (don't pay for off-topic generations).

**Follow-up 2: Frameworks for guardrails — build or buy?**
Options: NeMo Guardrails (NVIDIA), Guardrails AI, Llama Guard (content moderation), or roll your own with classifiers + rules. For most apps, a combination: a content-moderation model for harmful content, custom rules for scope/PII, and faithfulness checks for grounding. Start simple (system prompt + basic input/output checks), add sophistication where you see real failures.

---

### Q18. RAG vs fine-tuning vs prompting — how do you choose?

**Answer:**

**Prompting (in-context learning):**
- Put instructions/examples/data directly in the prompt.
- No training. Instant. Cheapest to iterate.
- Limited by context window; pay per token every call.
- **Use for:** most tasks. Start here.

**RAG:**
- Retrieve relevant knowledge at query time, inject into prompt.
- Knowledge is up-to-date (update the index), attributable, large.
- **Use for:** accessing private/current/large knowledge bases. Q&A over documents.

**Fine-tuning:**
- Train the model on your data to adjust its weights.
- Teaches *style, format, behavior, or specialized skills* — not facts (facts are better in RAG).
- Expensive to train, requires data, but can produce a smaller/cheaper model that performs a specific task well.
- **Use for:** consistent output format/style, domain-specific language, narrow tasks done at high volume, reducing prompt length (bake instructions into weights).

**Decision framework:**
| Need | Approach |
|------|----------|
| Access private/current facts | RAG |
| Consistent format/style/tone | Fine-tuning |
| One-off or low-volume task | Prompting |
| Specialized skill/behavior | Fine-tuning |
| Reduce per-call cost at high volume | Fine-tuning (smaller model) |
| Most things | Prompting, then RAG if knowledge-bound |

**They combine:** fine-tune for behavior + RAG for knowledge + prompting for the specific request. A fine-tuned model that follows your format, fed retrieved context via RAG, with a task-specific prompt.

**Worked example — your finance bot:**
- **RAG** for the financial data (facts, current, private). ✓ Primary approach.
- **Prompting** for the task structure and grounding rules. ✓
- **Fine-tuning** only if you needed consistent specialized output (e.g., a specific report format) at high volume — probably unnecessary for a chatbot.

**Follow-up 1: Why is fine-tuning bad for injecting facts?**
Fine-tuning adjusts weights toward patterns in training data, but it doesn't reliably store specific facts — and the model can still hallucinate, mixing fine-tuned facts with parametric knowledge. Updating a fact means retraining. RAG stores facts in a retrievable index — update by changing a document, attributable, no hallucination on the facts themselves. Facts → RAG; behavior → fine-tuning.

**Follow-up 2: When is fine-tuning clearly worth the cost?**
High-volume, narrow tasks where a fine-tuned small model matches a prompted large model at a fraction of the per-call cost. Example: classifying support tickets — fine-tune a small model, run it millions of times cheaply, vs prompting a large model each time. Also when you need a very specific output style/format that's hard to achieve reliably with prompting. For low-volume or evolving tasks, prompting + RAG is more flexible.

---

### Q19. What is prompt caching and why does it matter for RAG/agents?

**Answer:**
**Prompt caching** lets you cache a static prefix of your prompt so that repeated calls reusing that prefix are much cheaper and faster — you don't pay full price to reprocess the same tokens.

**Why it's huge for RAG and agents:**
- RAG prompts have a large static portion: system instructions, few-shot examples, retrieved context. Across multiple turns or similar queries, much of this repeats.
- Agents make many LLM calls in a loop, each re-sending the system prompt + tool definitions + accumulated context. Caching the stable prefix saves enormously.

**How it works (Anthropic/Claude):**
- Mark a prefix as cacheable with a cache breakpoint.
- The first call writes the cache (slight extra cost); subsequent calls within the cache lifetime read it at a large discount (cache reads are a fraction of input token cost) and lower latency.
- The cache has a TTL (e.g., 5 minutes for Anthropic's default).

**Code — prompt caching with Claude:**
```python
response = client.messages.create(
    model="claude-opus-4-7",
    max_tokens=1024,
    system=[
        {
            "type": "text",
            "text": LONG_SYSTEM_PROMPT_AND_INSTRUCTIONS,
            "cache_control": {"type": "ephemeral"},   # cache this prefix
        },
        {
            "type": "text",
            "text": retrieved_context,                 # also cacheable
            "cache_control": {"type": "ephemeral"},
        },
    ],
    messages=[{"role": "user", "content": user_query}],   # only this varies
)
```

**Best practices:**
- Put **stable content first** (system prompt, examples, large context), variable content (the user query) last.
- Order matters — the cache matches a *prefix*, so anything before the cache breakpoint must be identical across calls.
- Reuse within the TTL window — design conversation flows to hit the cache while it's warm.

**Impact:** for an agent loop or multi-turn chat over the same documents, prompt caching can cut input costs by 80-90% and reduce latency, because the bulk of the prompt is cached.

**Follow-up 1: What invalidates the cache?**
Any change to the cached prefix or anything before it. If you change one token in the system prompt, the cache misses. The cache also expires after its TTL (5 minutes for Anthropic's ephemeral cache, refreshed on each hit). So keep cached content stable and reuse it promptly. Reordering content or inserting variable data before the cache breakpoint defeats it.

**Follow-up 2: How does this change how you structure RAG prompts?**
Structure for cache reuse: stable system prompt and instructions first (cached), then retrieved context (cached if the same context is reused across a conversation), then the variable user query last (not cached). For a multi-turn chat about the same documents, the documents stay cached across turns. This is a real architectural consideration — design your prompt layout to maximize the cacheable prefix.

---

### Q20. What are the most common production gotchas for LLM applications?

**Answer:**

1. **No evaluation.** Tuning RAG/prompts by vibes. Build an eval set; measure every change. You can't improve what you don't measure.

2. **Ignoring cost.** LLM calls add up fast — an agent loop with 10 iterations × large context × expensive model = surprising bills. Monitor token usage, use caching, right-size models.

3. **No prompt caching.** Re-sending the same system prompt + context every call. Massive waste for RAG/agents.

4. **Hallucination unmanaged.** No grounding, no "I don't know" permission, no faithfulness check. The model confidently makes things up, especially numbers.

5. **Prompt injection unhandled.** User input or retrieved docs override your instructions. Delimiters, treat output as untrusted, limit tool privileges.

6. **Stale RAG index.** Documents updated but vectors not re-embedded. Answers reflect old data. Need an update pipeline.

7. **Bad chunking.** Chunks too big (diluted) or too small (no context) or split mid-idea. Ruins retrieval regardless of model quality.

8. **No streaming.** Users stare at a spinner for 10 seconds. Stream for perceived responsiveness.

9. **Letting the LLM do math.** Numerical errors. Compute in code; have the model format, not calculate.

10. **No tenant isolation in retrieval.** Multi-tenant RAG that retrieves across tenants = data leak. Metadata-filter by tenant on every search (your pgvector `WHERE tenant_id`).

11. **Unbounded agent loops.** Agent gets stuck, loops forever, burns money. Step limits and timeouts.

12. **No fallback when the LLM API is down/slow.** LLM providers have outages and latency spikes. Timeouts, retries with backoff, graceful degradation.

13. **Logging full prompts/responses with PII.** Sensitive data in logs. Redact.

14. **Temperature too high for factual tasks.** Creativity = hallucination for facts. Low temperature for retrieval/extraction.

15. **Not versioning prompts.** Prompt changes are code changes — version them, test them, roll back if quality regresses.

**Follow-up 1: How do you handle LLM API outages and latency spikes gracefully?**
Timeouts on every call, retries with exponential backoff for transient errors, circuit breaker to fail fast when the provider is down (section 07 Q10), and a fallback path: cached response, a cheaper/alternate model, or a graceful "I'm having trouble right now" message. For critical paths, multi-provider fallback (Claude → fallback to another). Treat the LLM as an unreliable external dependency, because it is.

**Follow-up 2: How do you safely roll out a prompt change?**
Treat prompts as code: version control, eval suite, staged rollout. Run the new prompt against your eval set, compare metrics to the current prompt. A/B test in production (new prompt on a fraction of traffic), monitor quality/cost/latency. Keep the ability to roll back instantly (feature flag the prompt version). A "small" prompt tweak can silently degrade quality — measure before full rollout.

---

*End of section 14. Next: Testing (10 questions).*
