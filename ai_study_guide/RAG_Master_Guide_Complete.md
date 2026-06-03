# The Complete RAG Master Guide
### Everything You Need to Learn RAG and Pass Any RAG Interview

> **A single source of truth covering RAG from first principles to enterprise architecture.** Read it linearly to learn, or jump to any section as a reference. Every concept includes the *why*, working code, and the interview angle.

**Scope:** Fundamentals → Implementation → Optimization → Agents → Enterprise Scale → Interview Mastery
**Audience:** Anyone from first-time learner to Principal Architect
**How to use:** Part 1-2 builds understanding. Part 3 covers advanced techniques. Part 4 is interview prep. Part 5 is system design. Part 6 is your quick reference.

---

## MASTER TABLE OF CONTENTS

**PART 1 — FOUNDATIONS**
- [1.1 What RAG Is and the Problem It Solves](#11-what-rag-is-and-the-problem-it-solves)
- [1.2 The Three-Step Process](#12-the-three-step-process)
- [1.3 Embeddings Explained](#13-embeddings-explained)
- [1.4 Chunking Explained](#14-chunking-explained)
- [1.5 Retrieval Explained](#15-retrieval-explained)
- [1.6 Critical Distinctions](#16-critical-distinctions)
- [1.7 Common Misconceptions](#17-common-misconceptions)

**PART 2 — BUILDING RAG**
- [2.1 Your First RAG System (Complete Code)](#21-your-first-rag-system-complete-code)
- [2.2 Parameters and Cost](#22-parameters-and-cost)
- [2.3 Diagnosing Failures](#23-diagnosing-failures)
- [2.4 Advanced Chunking](#24-advanced-chunking)
- [2.5 Reranking](#25-reranking)
- [2.6 Hybrid Search](#26-hybrid-search)
- [2.7 Measuring Quality](#27-measuring-quality)
- [2.8 Agents and Tool Use](#28-agents-and-tool-use)
- [2.9 Enterprise Scaling Code](#29-enterprise-scaling-code)

**PART 3 — ADVANCED TECHNIQUES**
- [3.1 Multi-Hop Retrieval](#31-multi-hop-retrieval)
- [3.2 Query Expansion and Rewriting](#32-query-expansion-and-rewriting)
- [3.3 Semantic Routing](#33-semantic-routing)
- [3.4 Knowledge Graph Integration](#34-knowledge-graph-integration)
- [3.5 Context Window Optimization](#35-context-window-optimization)
- [3.6 Advanced Optimization (Query, Doc, Dynamic-K)](#36-advanced-optimization)
- [3.7 Integration Patterns](#37-integration-patterns)
- [3.8 Emerging Techniques](#38-emerging-techniques)

**PART 4 — THEORY & EVALUATION**
- [4.1 How LLMs Work (Token Prediction)](#41-how-llms-work)
- [4.2 Transformer Attention](#42-transformer-attention)
- [4.3 Information Retrieval Theory](#43-information-retrieval-theory)
- [4.4 Embedding Geometry](#44-embedding-geometry)
- [4.5 Evaluation Frameworks (RAGAS, A/B)](#45-evaluation-frameworks)
- [4.6 Failure Diagnosis Methodology](#46-failure-diagnosis-methodology)

**PART 5 — SYSTEM DESIGN & DOMAINS**
- [5.1 Architecture Patterns by Scale](#51-architecture-patterns-by-scale)
- [5.2 Design: 10K Documents](#52-design-10k-documents)
- [5.3 Design: 10K Concurrent Users](#53-design-10k-concurrent-users)
- [5.4 Design: 100K+ Documents (Scaling)](#54-design-100k-documents)
- [5.5 Multi-Region Architecture](#55-multi-region-architecture)
- [5.6 Cost Optimization at Scale](#56-cost-optimization-at-scale)
- [5.7 Hallucination Prevention](#57-hallucination-prevention)
- [5.8 Production Patterns (Versioning, Caching, Failover)](#58-production-patterns)
- [5.9 Domain-Specific RAG (Legal, Medical, Financial, Code)](#59-domain-specific-rag)

**PART 6 — INTERVIEW MASTERY**
- [6.1 Question Bank by Level (25 Questions)](#61-question-bank-by-level)
- [6.2 Common Interview Mistakes](#62-common-interview-mistakes)
- [6.3 Interview Strategy Framework](#63-interview-strategy-framework)
- [6.4 Final Checklist](#64-final-checklist)

**PART 7 — QUICK REFERENCE**
- [7.1 API Cheat Sheet](#71-api-cheat-sheet)
- [7.2 Parameters & Cost Tables](#72-parameters--cost-tables)
- [7.3 Troubleshooting Guide](#73-troubleshooting-guide)
- [7.4 Glossary](#74-glossary)

---
---

# PART 1 — FOUNDATIONS

## 1.1 What RAG Is and the Problem It Solves

**RAG = Retrieval-Augmented Generation.** It connects an LLM to an external knowledge source so the model answers using *retrieved facts* instead of relying solely on what it memorized during training.

### The Core Problem

LLMs are trained on a fixed snapshot of data with a knowledge cutoff. They cannot reliably:
- Access current information (events after training)
- Know proprietary or internal data (your company's docs)
- Guarantee factual accuracy on niche topics
- Update knowledge without expensive retraining

When asked about something outside their training, they **hallucinate** — generate confident, plausible, but false answers.

```
WITHOUT RAG:
User: "What was our company's Q3 revenue?"
LLM:  "Based on industry trends, approximately $50 million."  ← guessing

WITH RAG:
User: "What was our company's Q3 revenue?"
System retrieves → "Q3 2024 Financial Report: Revenue = $47.3M"
LLM:  "Your Q3 2024 revenue was $47.3M."  ← factual, grounded
```

### The Key Insight

> LLMs are excellent at **reading and reasoning**, but poor at **recalling facts they never trained on.** RAG plays to the model's strength: give it the right document, and it synthesizes a great answer.

### Why Not Just Fine-Tune?

This is a classic interview follow-up. The honest answer:

| | RAG | Fine-Tuning |
|---|---|---|
| **Updates** | Instant (add/remove docs) | Requires retraining |
| **Cost** | Low (no training) | High (GPU hours) |
| **Traceability** | Yes (cite sources) | No (knowledge is opaque) |
| **Best for** | Facts, current info, proprietary data | Style, format, behavior, tone |

**Rule of thumb:** Use RAG for *knowledge*, fine-tuning for *behavior*. They're complementary, not competing.

---

## 1.2 The Three-Step Process

### Step 1 — Retrieval
Search your document collection for information relevant to the user's question.

```
Query: "How do I return a digital product?"
   ↓ (search by meaning)
Relevant chunks: ["Digital Products" section, "Returns Policy" section]
```

### Step 2 — Augmentation
Inject the retrieved documents into the prompt as context.

```
[CONTEXT]
Digital Products: 7-day return window. Not returnable if downloaded or
license key used.
---
Returns Policy: We accept returns for unused items in original packaging.

[QUESTION]
How do I return a digital product?
```

### Step 3 — Generation
The LLM reads the context and synthesizes an answer.

```
Output: "Digital products can be returned within 7 days, but only if they
haven't been downloaded and the license key hasn't been used."
```

That's the entire RAG loop: **Retrieve → Augment → Generate.** Everything else in this guide is making each of those three steps better, faster, cheaper, or more reliable.

---

## 1.3 Embeddings Explained

**What:** An embedding converts text into a vector — a list of numbers (typically ~1,536 dimensions) that represents the text's *meaning*.

**Why it works:** The embedding model was trained on billions of text pairs so that semantically similar texts produce similar vectors.

```
"Paris is the capital of France"  →  [0.023, -0.091, 0.456, ..., 0.234]
"France's capital is Paris"       →  [0.025, -0.088, 0.451, ..., 0.231]
                                       └── nearly identical vectors

cosine_similarity = 0.94  →  these mean almost the same thing
```

### Why This Beats Keyword Search

Keyword search fails on synonyms: searching "return policy" misses a document titled "refund procedures." Embeddings capture meaning, so "How do I reset my password?" retrieves a doc about "recover account access" even with zero shared keywords.

### Cosine Similarity

The math for comparing two embeddings:

```python
similarity = (v1 · v2) / (|v1| × |v2|)
# 1.0 = identical meaning,  0.5 = related,  0.0 = unrelated
```

It measures the *angle* between vectors, ignoring magnitude — which is exactly what we want for comparing meaning.

---

## 1.4 Chunking Explained

**What:** Splitting documents into smaller segments before indexing.

**Why it matters** — sending whole documents to an LLM is expensive, slow, and noisy:

```
WHOLE 100-page manual:   50,000 tokens → $1.50/query → 5s → noisy, unfocused
CHUNKED (top-3 of 500):   1,500 tokens → $0.002/query → 1s → focused, accurate
                          └── 100× cheaper, 5× faster, higher quality
```

### Three Strategies

| Strategy | How | Pros | Cons |
|----------|-----|------|------|
| **Fixed-size** | Split every N chars | Simple, predictable | Ignores structure, may split mid-sentence |
| **Semantic** | Split at paragraph/section | Coherent, respects structure | Variable sizes, needs parsing |
| **Recursive** | Try large boundaries, split smaller if needed | Works on any structure | More complex |

**Starting defaults:** 500-char chunks, 100-char overlap. The overlap preserves context across chunk boundaries so an answer split between two chunks isn't lost.

> **The trade-off to internalize:** Small chunks = precise but may lose context. Large chunks = rich context but more noise. You tune this against your actual data.

---

## 1.5 Retrieval Explained

The retrieval process, step by step:

```
1. Embed query        → convert question to a vector
2. Similarity search  → compare against every chunk vector (cosine similarity)
3. Rank               → sort chunks by similarity score
4. Select top-k       → return the k most similar (typically 3-5)
```

**Why top-k instead of top-1?** The single most similar chunk might not contain the full answer. Top-3 gives the LLM enough context without drowning it in noise. Too high (top-20) reintroduces the noise problem.

---

## 1.6 Critical Distinctions

### Similarity vs. Relevance — the most important distinction in RAG

These are **not the same thing**, and confusing them is the root of most retrieval failures.

- **Similarity:** How alike are two texts? (measured by embedding cosine)
- **Relevance:** Does this text actually *answer the question*? (requires judgment)

```
Query: "Can I return a digital product?"

Doc A: "Digital products cannot be returned after download"
       Similarity 0.95  |  Relevance HIGH  ✓ (directly answers)

Doc B: "Physical products have a 30-day return window"
       Similarity 0.72  |  Relevance LOW   ✗ (wrong product type)
```

A naive system ranks by similarity and might surface Doc B. **Reranking** (Section 2.5) re-sorts by relevance — this is *why* it improves quality.

### Knowledge vs. Hallucination

- **Knowledge:** Facts seen many times in training → confident, usually correct.
- **Hallucination:** Facts never seen (proprietary, current) → the model guesses fluently.

RAG converts a hallucination-prone question into a reading-comprehension task by supplying the facts.

---

## 1.7 Common Misconceptions

| Misconception | Reality |
|---------------|---------|
| "RAG makes the LLM smarter" | It makes it *more informed*, not smarter. Reasoning ability is unchanged. |
| "RAG eliminates hallucination" | It *reduces* it. The model can still misread or over-extend the context. |
| "Bigger documents = better answers" | Bigger = more noise. Focused chunks win. |
| "Retrieve more docs = better" | More docs = more confusion. Fewer, more relevant docs win. |
| "Better embeddings fix everything" | Embeddings are one lever. Chunking, reranking, and prompting matter just as much. |

---
---

# PART 2 — BUILDING RAG

## 2.1 Your First RAG System (Complete Code)

A full, runnable RAG system. This is your foundation — every later technique extends this class.

### Setup

```bash
pip install anthropic openai python-dotenv
```

`.env` file:
```
ANTHROPIC_API_KEY=sk-ant-...
OPENAI_API_KEY=sk-...
```

### Complete Code

```python
"""
Foundational RAG System — runnable end to end.
Chunk → Embed → Retrieve → Generate.
"""
import os
import math
from typing import List
from dotenv import load_dotenv
from anthropic import Anthropic
from openai import OpenAI

load_dotenv()
anthropic_client = Anthropic(api_key=os.getenv("ANTHROPIC_API_KEY"))
openai_client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))

# ---------- CHUNKING ----------
def chunk_text(text: str, chunk_size: int = 500, overlap: int = 100) -> List[str]:
    """Split text into overlapping chunks. Overlap preserves context at boundaries."""
    chunks, start = [], 0
    while start < len(text):
        end = min(start + chunk_size, len(text))
        chunk = text[start:end].strip()
        if chunk:
            chunks.append(chunk)
        start += chunk_size - overlap
    return chunks

# ---------- EMBEDDINGS ----------
def get_embedding(text: str, model: str = "text-embedding-3-small") -> List[float]:
    """Convert text to a ~1,536-dim vector representing its meaning."""
    resp = openai_client.embeddings.create(input=text, model=model)
    return resp.data[0].embedding

def cosine_similarity(v1: List[float], v2: List[float]) -> float:
    """Angle between vectors. 1.0 = same meaning, 0.0 = unrelated."""
    dot = sum(a * b for a, b in zip(v1, v2))
    m1 = math.sqrt(sum(a * a for a in v1))
    m2 = math.sqrt(sum(b * b for b in v2))
    return dot / (m1 * m2) if m1 and m2 else 0.0

# ---------- RAG SYSTEM ----------
class RAGSystem:
    """Chunk → Embed → Retrieve → Generate."""

    def __init__(self):
        self.chunks: List[str] = []
        self.embeddings: List[List[float]] = []
        self.metadata: List[dict] = []

    def add_document(self, text: str, doc_name: str = "document"):
        chunks = chunk_text(text, chunk_size=500, overlap=100)
        for i, chunk in enumerate(chunks):
            self.chunks.append(chunk)
            self.embeddings.append(get_embedding(chunk))
            self.metadata.append({"doc_name": doc_name, "chunk_index": i})
        print(f"Stored {len(chunks)} chunks from '{doc_name}'")

    def retrieve(self, query: str, top_k: int = 3) -> List[str]:
        if not self.chunks:
            return []
        q_emb = get_embedding(query)
        scored = [
            {"chunk": self.chunks[i], "sim": cosine_similarity(q_emb, emb)}
            for i, emb in enumerate(self.embeddings)
        ]
        scored.sort(key=lambda x: x["sim"], reverse=True)
        return [s["chunk"] for s in scored[:top_k]]

    def query(self, user_query: str, top_k: int = 3) -> str:
        # 1. RETRIEVE
        chunks = self.retrieve(user_query, top_k=top_k)
        if not chunks:
            return "No relevant documents found."
        # 2. AUGMENT
        context = "\n\n---\n\n".join(chunks)
        system_prompt = (
            "You are a helpful assistant. Answer ONLY from the provided context. "
            "If the answer isn't in the context, say so clearly."
        )
        user_message = f"Context:\n\n{context}\n\n---\n\nQuestion: {user_query}\n\nAnswer:"
        # 3. GENERATE
        resp = anthropic_client.messages.create(
            model="claude-opus-4-20250805",
            max_tokens=1024,
            system=system_prompt,
            messages=[{"role": "user", "content": user_message}],
        )
        return resp.content[0].text


if __name__ == "__main__":
    doc = """
    COMPANY RETURN POLICY
    Physical products: 30 days from purchase. Digital products: 7 days
    (non-refundable if downloaded). Original packaging required.
    Defective items: full refund regardless of timeframe.
    Sale items: 14-day window. Clearance: final sale.
    """
    rag = RAGSystem()
    rag.add_document(doc, "Return Policy")
    for q in ["Can I return a digital product?", "What if the item is defective?"]:
        print(f"\nQ: {q}\nA: {rag.query(q)}")
```

---

## 2.2 Parameters and Cost

```python
chunk_size = 500     # balance context vs. focus
overlap    = 100     # preserve context at boundaries
top_k      = 3       # documents retrieved per query
max_tokens = 1024    # response length cap
temperature = 0.3    # for RAG, keep LOW (factual, not creative)
```

**Cost per query (approximate):**
```
Query embedding:           ~$0.0002
Claude (500 in, 200 out):  ~$0.0018
                           ─────────
Total:                     ~$0.002  (less than a penny)

1,000 queries/day  →  ~$60/month
```

---

## 2.3 Diagnosing Failures

When RAG gives a bad answer, isolate the failure with this decision tree. **This framework is gold in interviews** — it shows systematic debugging.

```
1. Is the info actually IN the documents?
     NO  → DATA problem (need better source data)
     YES ↓
2. Can embeddings find it? (high similarity score?)
     NO  → EMBEDDING problem (wrong model / domain mismatch)
     YES ↓
3. Is it in the top-k results?
     NO  → RETRIEVAL RANKING problem (rerank / raise top-k)
     YES ↓
4. Does the LLM use it correctly?
     NO  → GENERATION problem (fix system prompt / temperature)
     YES → RAG WORKS ✓
```

Diagnostic helper:

```python
def diagnose(rag: RAGSystem, query: str, top_k: int = 10):
    """Print similarity scores for every chunk to see what retrieval is doing."""
    q_emb = get_embedding(query)
    scored = sorted(
        [(cosine_similarity(q_emb, e), rag.chunks[i]) for i, e in enumerate(rag.embeddings)],
        reverse=True,
    )
    for rank, (sim, chunk) in enumerate(scored[:top_k], 1):
        flag = "⭐ retrieved" if rank <= 3 else ""
        print(f"{rank:2d}. sim={sim:.3f} {flag}  {chunk[:50]}...")
```

---

## 2.4 Advanced Chunking

### Semantic Chunking (respects structure)

```python
def chunk_by_sections(text: str, marker: str = "\n\n") -> List[str]:
    return [s.strip() for s in text.split(marker) if s.strip()]
```
Use for documents with clear paragraph/section boundaries.

### Recursive Chunking (works on any structure)

```python
def chunk_recursively(text: str, max_size: int = 1000,
                      separators: List[str] = None) -> List[str]:
    """Try large boundaries first; split smaller only when needed."""
    if separators is None:
        separators = ["\n\n", "\n", ". ", " "]  # large → small

    def _split(t, i=0):
        if i >= len(separators):
            return [t] if t else []
        out = []
        for part in t.split(separators[i]):
            if len(part) < max_size:
                out.append(part)
            else:
                out.extend(_split(part, i + 1))
        return [p for p in out if p.strip()]

    return _split(text)
```
Use for complex, multi-level documents (manuals, books, mixed content).

---

## 2.5 Reranking

Reranking separates **relevance** from **similarity** (recall Section 1.6). Retrieve a wide net cheaply with embeddings, then re-sort precisely.

```python
import json

class RAGWithReranking(RAGSystem):
    def rerank(self, query: str, candidates: List[str], top_k: int = 3) -> List[str]:
        listing = "\n\n".join(f"[{i+1}] {c[:200]}" for i, c in enumerate(candidates))
        prompt = (
            f"Query: {query}\n\nRank these documents by RELEVANCE (10=directly answers, "
            f"1=barely relevant):\n\n{listing}\n\n"
            f'Respond ONLY with JSON like {{"1": 8, "2": 3}}.'
        )
        resp = anthropic_client.messages.create(
            model="claude-opus-4-20250805", max_tokens=300,
            messages=[{"role": "user", "content": prompt}],
        )
        try:
            scores = json.loads(resp.content[0].text)
            ranked = sorted(enumerate(candidates),
                            key=lambda x: scores.get(str(x[0] + 1), 0), reverse=True)
            return [c for _, c in ranked[:top_k]]
        except json.JSONDecodeError:
            return candidates[:top_k]

    def query_reranked(self, query: str, retrieve_k: int = 10, final_k: int = 3) -> str:
        candidates = self.retrieve(query, top_k=retrieve_k)   # wide net
        best = self.rerank(query, candidates, top_k=final_k)  # precise sort
        context = "\n\n---\n\n".join(best)
        resp = anthropic_client.messages.create(
            model="claude-opus-4-20250805", max_tokens=1024,
            system="Answer from the context only.",
            messages=[{"role": "user", "content": f"Context:\n{context}\n\nQ: {query}\nA:"}],
        )
        return resp.content[0].text
```

**Trade-off:** +1 LLM call (~$0.001, +200ms latency) for a typical 10-15% precision gain. Worth it when accuracy matters more than cost/speed.

---

## 2.6 Hybrid Search

Combine semantic search (great for meaning) with keyword search (great for exact terms, codes, names).

```python
class HybridRAG(RAGWithReranking):
    def keyword_search(self, query: str, top_k: int = 5) -> List[str]:
        terms = {w.lower() for w in query.split() if len(w) > 3}
        scored = []
        for chunk in self.chunks:
            hits = sum(1 for t in terms if t in chunk.lower())
            if hits:
                scored.append((hits, chunk))
        scored.sort(reverse=True)
        return [c for _, c in scored[:top_k]]

    def hybrid_retrieve(self, query: str, k_sem: int = 5, k_kw: int = 5) -> List[str]:
        semantic = set(self.retrieve(query, top_k=k_sem))
        keyword = set(self.keyword_search(query, top_k=k_kw))
        return list(semantic | keyword)   # union, deduped
```

**When to use:** Legal docs (exact phrase matching), product catalogs (SKUs, model numbers), any mix of conceptual and exact-match queries.

---

## 2.7 Measuring Quality

You can't improve what you don't measure.

```python
def precision(retrieved: List[str], relevant: List[str]) -> float:
    """Of retrieved docs, how many were relevant?"""
    r, ret = set(relevant), set(retrieved)
    return len(r & ret) / len(ret) if ret else 0.0

def recall(retrieved: List[str], relevant: List[str]) -> float:
    """Of all relevant docs, how many did we find?"""
    r, ret = set(relevant), set(retrieved)
    return len(r & ret) / len(r) if r else 1.0
```

Build a labeled test set of 50-100 query/expected-doc pairs and track precision@k, recall@k over time. (For answer-quality scoring and RAGAS, see Section 4.5.)

---

## 2.8 Agents and Tool Use

RAG becomes far more powerful when the LLM can *decide* to retrieve, calculate, or call other tools.

### Defining Tools

```python
tools = [
    {
        "name": "search_documents",
        "description": "Search the knowledge base for relevant information",
        "input_schema": {
            "type": "object",
            "properties": {"query": {"type": "string"}},
            "required": ["query"],
        },
    },
    {
        "name": "calculate",
        "description": "Evaluate a mathematical expression",
        "input_schema": {
            "type": "object",
            "properties": {"expression": {"type": "string"}},
            "required": ["expression"],
        },
    },
]

def execute_tool(name: str, inp: dict, rag: RAGSystem) -> str:
    if name == "search_documents":
        return "\n\n".join(rag.retrieve(inp["query"], top_k=3))
    if name == "calculate":
        try:
            return f"Result: {eval(inp['expression'])}"
        except Exception as e:
            return f"Error: {e}"
    return "Unknown tool"
```

### The Agent Loop

```python
class Agent:
    def __init__(self, rag: RAGSystem, max_iterations: int = 10):
        self.rag = rag
        self.max_iterations = max_iterations

    def run(self, task: str) -> str:
        messages = [{"role": "user", "content": task}]
        for _ in range(self.max_iterations):
            resp = anthropic_client.messages.create(
                model="claude-opus-4-20250805", max_tokens=2048,
                tools=tools, messages=messages,
                system="Solve the task step by step. Use tools when you need information.",
            )
            if resp.stop_reason == "end_turn":
                return next((b.text for b in resp.content if hasattr(b, "text")), "")
            calls = [b for b in resp.content if b.type == "tool_use"]
            if not calls:
                return next((b.text for b in resp.content if hasattr(b, "text")), "")
            messages.append({"role": "assistant", "content": resp.content})
            results = [
                {"type": "tool_result", "tool_use_id": c.id,
                 "content": execute_tool(c.name, c.input, self.rag)}
                for c in calls
            ]
            messages.append({"role": "user", "content": results})
        return "Max iterations reached."
```

> **⚠️ The #1 agent pitfall:** loops that never terminate. Always set `max_iterations`, define explicit stop conditions, and handle tool failures gracefully.

---

## 2.9 Enterprise Scaling Code

### Batch Embedding (100× fewer API calls)

```python
def batch_embed(texts: List[str], batch_size: int = 100) -> List[List[float]]:
    out = []
    for i in range(0, len(texts), batch_size):
        resp = openai_client.embeddings.create(
            input=texts[i:i + batch_size], model="text-embedding-3-small"
        )
        out.extend(item.embedding for item in resp.data)
    return out
# 1,000 texts: 1,000 calls (one-by-one) → 10 calls (batched)
```

### Embedding Cache

```python
class CachedEmbeddings:
    def __init__(self):
        self.cache = {}
    def get(self, text: str) -> List[float]:
        if text not in self.cache:
            self.cache[text] = get_embedding(text)
        return self.cache[text]
```

### Async High-Throughput

```python
import asyncio
from anthropic import AsyncAnthropic

aclient = AsyncAnthropic()

async def process_batch(queries: List[str]) -> List[str]:
    async def one(q: str) -> str:
        resp = await aclient.messages.create(
            model="claude-opus-4-20250805", max_tokens=512,
            messages=[{"role": "user", "content": q}],
        )
        return resp.content[0].text
    return await asyncio.gather(*[one(q) for q in queries])
```

### Cost-Routing (70% savings)

```python
def route_by_complexity(query: str, rag: RAGSystem) -> str:
    """Cheap model for simple queries, expensive for complex."""
    # classify cheaply with Haiku, then route — see Section 3.3 for full version
    ...
```
Full routing implementation in [Section 3.3](#33-semantic-routing).

---
---

# PART 3 — ADVANCED TECHNIQUES

## 3.1 Multi-Hop Retrieval

**Problem:** Some questions need information from *multiple* documents combined.

```
"What was the impact of the Q3 earnings announcement on our return policy?"
  needs: Q3 earnings (financial doc) + return policy (policy doc) + reasoning
```

A single retrieval pass can't connect these. Multi-hop retrieves iteratively, using each result to inform the next search.

```python
class MultiHopRAG(RAGSystem):
    def multi_hop_query(self, query: str, hops: int = 3) -> str:
        gathered, search_query = [], query
        for hop in range(hops):
            docs = self.retrieve(search_query, top_k=3)
            gathered.extend(docs)
            if hop < hops - 1:
                refine = (
                    f"Original question: {query}\n"
                    f"Gathered so far: {' '.join(gathered)[:1000]}\n"
                    "What ONE additional thing do we still need to know? "
                    "Reply with a single search query."
                )
                resp = anthropic_client.messages.create(
                    model="claude-opus-4-20250805", max_tokens=80,
                    messages=[{"role": "user", "content": refine}],
                )
                search_query = resp.content[0].text.strip()
        context = "\n\n---\n\n".join(set(gathered))
        resp = anthropic_client.messages.create(
            model="claude-opus-4-20250805", max_tokens=1024,
            messages=[{"role": "user",
                       "content": f"Sources:\n{context}\n\nAnswer: {query}"}],
        )
        return resp.content[0].text
```

**Use cases:** comparative analysis, timeline/causal questions, impact analysis.
**Interview angle:** "How would you answer 'Compare Q2 vs Q3 performance given market conditions'?" → multi-hop is the answer.

---

## 3.2 Query Expansion and Rewriting

**Problem:** The user's phrasing may retrieve poorly. "How do I return stuff?" is vague; "return policy physical products refund process" retrieves better.

```python
class QueryExpansionRAG(RAGSystem):
    def expand(self, query: str) -> List[str]:
        prompt = (
            f'User query: "{query}"\n'
            "Generate 3 alternative search queries emphasizing different aspects "
            'or terminology. Return JSON list ["q1","q2","q3"].'
        )
        resp = anthropic_client.messages.create(
            model="claude-3-5-haiku-20241022", max_tokens=200,
            messages=[{"role": "user", "content": prompt}],
        )
        try:
            return [query] + json.loads(resp.content[0].text)
        except Exception:
            return [query]

    def query_expanded(self, query: str) -> str:
        docs = set()
        for q in self.expand(query):
            docs.update(self.retrieve(q, top_k=3))
        context = "\n\n---\n\n".join(list(docs)[:5])
        resp = anthropic_client.messages.create(
            model="claude-opus-4-20250805", max_tokens=1024,
            messages=[{"role": "user",
                       "content": f"Context:\n{context}\n\nQ: {query}\nA:"}],
        )
        return resp.content[0].text
```

**Impact:** recall typically improves 15-30%. Cost: +1 cheap Haiku call.

---

## 3.3 Semantic Routing

**Problem:** Different query types deserve different strategies. A simple fact lookup shouldn't pay for the full rerank pipeline.

```python
class SemanticRoutingRAG(RAGWithReranking):
    def classify(self, query: str) -> str:
        prompt = (
            f'Query: "{query}"\nClassify as exactly one: '
            "FACTUAL, PROCEDURAL, ANALYTICAL, COMPARATIVE, RECOMMENDATION. "
            "Reply with only the label."
        )
        resp = anthropic_client.messages.create(
            model="claude-3-5-haiku-20241022", max_tokens=10,
            messages=[{"role": "user", "content": prompt}],
        )
        return resp.content[0].text.strip()

    def query_routed(self, query: str) -> str:
        qtype = self.classify(query)
        if qtype == "FACTUAL":               # cheap + fast
            docs, model, mt = self.retrieve(query, top_k=1), "claude-3-5-haiku-20241022", 100
        elif qtype == "PROCEDURAL":
            docs, model, mt = self.retrieve(query, top_k=3), "claude-opus-4-20250805", 512
        else:                                # ANALYTICAL/COMPARATIVE/RECOMMENDATION
            cand = self.retrieve(query, top_k=10)
            docs, model, mt = self.rerank(query, cand, 3), "claude-opus-4-20250805", 1024
        context = "\n\n---\n\n".join(docs)
        resp = anthropic_client.messages.create(
            model=model, max_tokens=mt,
            messages=[{"role": "user", "content": f"Context:\n{context}\n\nQ: {query}\nA:"}],
        )
        return resp.content[0].text
```

**Cost impact (weighted by typical traffic):**
```
40% FACTUAL  (Haiku, top-1):  $0.0005
30% PROCED.  (Opus,  top-3):  $0.002
20% ANALYT.  (Opus,  rerank): $0.004
10% COMPLEX  (Opus,  rerank): $0.010
Weighted average: ~$0.0023/query  vs  $0.005 flat  →  ~54% savings
```

---

## 3.4 Knowledge Graph Integration

Pure text retrieval misses *relationships*. A knowledge graph encodes them explicitly:

```
[Digital Products] --applies-to--> [7-day window]
[Return Policy]    --requires-->   [Original Packaging]
[Defective Items]  --exception-->  [Return Window]
```

```python
class GraphRAG(RAGSystem):
    def __init__(self):
        super().__init__()
        self.graph = {}   # entity -> set of related entities/docs

    def graph_aware_retrieve(self, query: str, top_k: int = 3) -> List[str]:
        seed = self.retrieve(query, top_k=2)          # semantic seed
        related = set()
        for doc in seed:
            related.update(self.graph.get(doc, set())) # follow edges
        return list(set(seed) | related)[:top_k]
```

**When to use:** legal/medical/financial corpora where relationships (precedents, contraindications, dependencies) carry the meaning.

---

## 3.5 Context Window Optimization

Claude Opus offers a **200K-token** window. Strategic allocation:

```
System prompt        ~500 tokens
User query           ~100 tokens
Output reserve     ~1,000 tokens
Buffer/overhead    ~5,000 tokens
─────────────────────────────
Available for docs ~193K tokens  →  room for 30-50+ chunks
```

With long context, the strategy *shifts*:

```python
class LongContextRAG(RAGWithReranking):
    def query_abundant(self, query: str) -> str:
        # retrieve broadly; light rerank; let the model synthesize
        candidates = self.retrieve(query, top_k=50)
        best = self.rerank(query, candidates, top_k=30)
        primary, supporting = best[:5], best[5:]
        context = (
            "PRIMARY SOURCES:\n" + "\n".join(f"- {c[:300]}" for c in primary) +
            "\n\nSUPPORTING:\n" + "\n".join(f"- {c[:150]}" for c in supporting)
        )
        resp = anthropic_client.messages.create(
            model="claude-opus-4-20250805", max_tokens=2048,
            system="Synthesize ALL provided sources into a comprehensive answer.",
            messages=[{"role": "user", "content": f"{context}\n\nQ: {query}"}],
        )
        return resp.content[0].text
```

> **The mindset shift:** with small windows you optimize *retrieval precision*. With large windows you optimize *organization and synthesis* — retrieval failures drop because you can afford broad coverage, but you must structure the context so the model knows what's primary vs. supporting.

---

## 3.6 Advanced Optimization

### Query Optimization (rewrite before retrieving)

```python
def optimize_query(query: str) -> str:
    prompt = (
        f'Rewrite for optimal retrieval (be specific, use domain terms, '
        f'avoid vague words like "stuff"): "{query}" →'
    )
    resp = anthropic_client.messages.create(
        model="claude-3-5-haiku-20241022", max_tokens=60,
        messages=[{"role": "user", "content": prompt}],
    )
    return resp.content[0].text.strip()
# "How do I return stuff?" → "return policy physical products refund process"
# Typical precision@3 gain: 65% → 78%
```

### Document Optimization (format for retrieval)

Prepend a summary + keywords + FAQ to each document. The summary helps the model judge relevance fast; keywords improve semantic match; the FAQ catches common phrasings directly.

### Dynamic Top-K (don't always retrieve 3)

```python
class AdaptiveTopK(RAGSystem):
    def adaptive_retrieve(self, query: str) -> List[str]:
        q = get_embedding(query)
        scored = sorted(
            ((cosine_similarity(q, e), self.chunks[i]) for i, e in enumerate(self.embeddings)),
            reverse=True,
        )
        out = []
        for sim, chunk in scored:
            if sim > 0.8:                       # high confidence: take it
                out.append(chunk)
            elif sim > 0.6 and len(out) < 3:    # medium: cap at 3
                out.append(chunk)
            else:
                break
        return out or [scored[0][1]]            # always return at least one
```
Avoids feeding low-quality chunks; saves cost when few are truly relevant.

---

## 3.7 Integration Patterns

Pure RAG is rarely standalone in production. Common combinations:

**RAG + Structured Data** — route numeric/date/category questions to a database query, conceptual questions to text retrieval, then merge both into the context.

**RAG + Real-Time APIs** — detect when a question needs live data (prices, weather, news), fetch it, and combine with the static knowledge base.

```python
class HybridSourceRAG(RAGSystem):
    def query_hybrid(self, query: str) -> str:
        needs_live = self._needs_live_data(query)         # cheap Haiku classify
        text = self.retrieve(query, top_k=3)
        live = self._fetch_live(query) if needs_live else ""
        context = f"Knowledge base:\n{chr(10).join(text)}\n\nLive data:\n{live}"
        resp = anthropic_client.messages.create(
            model="claude-opus-4-20250805", max_tokens=1024,
            messages=[{"role": "user", "content": f"{context}\n\nQ: {query}"}],
        )
        return resp.content[0].text
    def _needs_live_data(self, q): ...   # implement classifier
    def _fetch_live(self, q): ...        # implement API calls
```

---

## 3.8 Emerging Techniques

**Long-Context Models** — covered in 3.5. The trend is toward retrieving more and synthesizing rather than retrieving narrowly.

**Multimodal RAG** — index and retrieve images, tables, and diagrams alongside text. Modern Claude models can reason over images directly, so retrieval can return visual context.

**Agentic RAG** — the model decides *what* to retrieve and *when*, choosing among tools (knowledge base, web, calculator) dynamically. This is RAG + the agent loop from Section 2.8, with retrieval as one of several tools the agent can invoke.

```python
class AgenticRAG(RAGSystem):
    def agentic_query(self, query: str) -> str:
        # identical to the Agent loop in 2.8, with search_documents,
        # search_web, and calculate exposed as tools — the model
        # chooses which to call and in what order.
        return Agent(self).run(query)
```

---
---

# PART 4 — THEORY & EVALUATION

## 4.1 How LLMs Work

LLMs predict the next token given previous tokens: `P(next_token | previous_tokens)`.

```
Input:  "The capital of France is"
Model computes a probability distribution over the vocabulary:
   P("Paris")  = 0.95
   P("France") = 0.02
   P("London") = 0.001
Output: "Paris"  (highest probability)
```

**Why context helps:** Without context, "What's our Q3 revenue?" has only vague priors → a guess. With retrieved context, the model conditions on `P(revenue | "Q3 Report: $47.3M")` → a confident, factual answer. RAG literally reshapes the probability distribution by adding evidence to the input.

**Temperature** controls randomness in sampling: 0.0 deterministic, 1.0 balanced, 2.0 wild. **For RAG, use 0.1-0.5** — you want faithful, factual output, not creativity.

---

## 4.2 Transformer Attention

You don't implement transformers, but understanding attention explains *why RAG works* and *why context windows are limited*.

```
Attention = weighted sum of Values, weighted by Query-Key relevance.

Query (Q): what am I looking for?
Key   (K): what information is available?
Value (V): the information itself

relevance = (Q · K) / sqrt(dimension)
attention = softmax(relevance) × V
```

**Implications for RAG:**
- The model can attend to retrieved context — that's how injected docs influence the answer.
- More context = more to attend to, but attention *diffuses* across many tokens. Too much irrelevant context dilutes focus → why we limit/rerank top-k even with big windows.

---

## 4.3 Information Retrieval Theory

There are four search paradigms; production systems blend them.

| Type | How | Strength | Weakness | Best for |
|------|-----|----------|----------|----------|
| **Keyword (BM25)** | TF-IDF term matching | Exact terms | Misses synonyms | Legal, exact phrases |
| **Semantic** | Embedding nearest-neighbor | Meaning, synonyms | Exact-match edge cases | Q&A, general search |
| **Lexical (exact)** | Inverted index | Precise codes/IDs | No meaning | Structured lookups |
| **Hybrid** | Combine the above | Best coverage | More complex | Production |

Hybrid ranking example: `score = 0.7 × semantic + 0.3 × bm25` (weights tuned to your data).

**Relevance ≠ Similarity** (the Section 1.6 distinction) is itself an IR principle — embeddings optimize for similarity, but the metric you actually care about is relevance, which is why reranking exists.

---

## 4.4 Embedding Geometry

Why embeddings capture meaning:

```
Famous property:  king − man + woman ≈ queen
Semantic relationships become vector arithmetic — learned automatically
from billions of examples via contrastive training.
```

**Cosine similarity** (`(v1·v2)/(|v1||v2|)`) measures angle, ignoring magnitude → robust for comparing meaning.

**Why ~1,536 dimensions?** Empirical sweet spot: too few (<100) loses information; too many (>10K) adds compute cost with diminishing returns.

---

## 4.5 Evaluation Frameworks

### RAGAS — four dimensions of RAG quality

| Dimension | Question it answers |
|-----------|---------------------|
| **Faithfulness** | Is the answer grounded in the retrieved context (no hallucination)? |
| **Answer Relevance** | Does the answer address the question asked? |
| **Context Relevance** | Was the retrieved context actually relevant? |
| **Context Utilization** | Did the model use the context (vs. ignore it)? |

```python
class RAGAS:
    def _score(self, prompt: str) -> float:
        r = anthropic_client.messages.create(
            model="claude-opus-4-20250805", max_tokens=10,
            messages=[{"role": "user", "content": prompt}],
        )
        try:
            return int(r.content[0].text.strip()) / 10.0
        except ValueError:
            return 0.0

    def faithfulness(self, context, answer):
        return self._score(f"Context:\n{context}\nAnswer:\n{answer}\n"
                           "Is the answer fully supported by the context? Score 0-10. Reply number only.")
    def answer_relevance(self, query, answer):
        return self._score(f"Q: {query}\nA: {answer}\nDoes the answer address the question? 0-10. Number only.")
    def context_relevance(self, query, context):
        return self._score(f"Q: {query}\nContext: {context}\nIs the context relevant? 0-10. Number only.")

    def evaluate(self, query, context, answer):
        f = self.faithfulness(context, answer)
        ar = self.answer_relevance(query, answer)
        cr = self.context_relevance(query, context)
        return {"faithfulness": f, "answer_relevance": ar,
                "context_relevance": cr, "overall": (f + ar + cr) / 3}
```

### A/B Testing with significance

```python
import math
def ab_test(queries, rag_a, rag_b, metric_fn):
    a = [metric_fn(q, rag_a.query(q)) for q in queries]
    b = [metric_fn(q, rag_b.query(q)) for q in queries]
    ma, mb, n = sum(a)/len(a), sum(b)/len(b), len(a)
    va = sum((x-ma)**2 for x in a)/n
    vb = sum((x-mb)**2 for x in b)/n
    se = math.sqrt((va+vb)/n)
    t = (mb-ma)/se if se else 0
    return {"mean_a": ma, "mean_b": mb,
            "improvement": (mb-ma)/ma if ma else 0,
            "significant": abs(t) > 1.96}   # ~95% confidence
```

**Best practices:** ≥50 queries, run 1-2 weeks (not minutes), report confidence not just point estimates, watch for regressions.

---

## 4.6 Failure Diagnosis Methodology

The 4-layer diagnosis (expanded from Section 2.3) — know this cold for interviews:

```
LAYER          SYMPTOM                          ROOT CAUSE               FIX
─────────────────────────────────────────────────────────────────────────────
DATA           Info not in any chunk            Missing source data      Add/fix docs
RETRIEVAL      Low similarity for all chunks    Wrong embedding model    Swap/fine-tune model
RANKING        Relevant chunk exists but not    Similarity ≠ relevance   Add reranking
               in top-k                                                  / raise k
GENERATION     Right context, wrong answer      Prompt/temperature       Tighten prompt,
                                                                         lower temp
SYNTHESIS      Multiple docs, fails to combine  Single-hop limitation    Multi-hop (3.1)
```

**When RAG makes things *worse*:** if retrieval surfaces irrelevant-but-confident context, the model may anchor on it and produce a worse answer than no-RAG. Mitigation: confidence thresholds — if top similarity < 0.5, decline or fall back rather than forcing an answer.

---
---

# PART 5 — SYSTEM DESIGN & DOMAINS

## 5.1 Architecture Patterns by Scale

| Scale | Pattern | Storage | Cost/mo | Latency |
|-------|---------|---------|---------|---------|
| 1-100 docs | In-memory | Python lists | <$1 | <500ms |
| 100-10K docs | Managed vector DB | Pinecone/Weaviate | ~$2-3K | <1s |
| 10K-100K docs | Distributed + cache | Vector cluster + Redis | ~$15-30K | <2s P99 |
| 100K+ docs, 1K+ users | Multi-region | Geo-distributed | $50-100K+ | <3s P99 |

The architecture doesn't fundamentally change with scale — you *add layers* (caching, distribution, queueing, monitoring) as load grows.

---

## 5.2 Design: 10K Documents

```
DATA SOURCES (files, DBs, APIs)
        │
   INDEXING PIPELINE (offline/batch)
   load → chunk → batch-embed(100) → store
        │
   VECTOR DATABASE  (Pinecone / Weaviate / Milvus)
   ~10K docs → 50-100K chunks → ~750 MB
        │
   CACHE LAYER (query + embedding + result cache)
        │
   RETRIEVAL ENGINE
   embed query (10ms) → search (50ms) → rerank top-10 (200ms) → top-3
        │
   LLM PIPELINE (build prompt → Claude → stream)
        │
   RESPONSE (answer + confidence + sources)
```

**Key decisions:**

| Decision | Choice | Why |
|----------|--------|-----|
| Chunk size | 500 chars, semantic | Balance context/focus |
| Retrieval | top-10 → rerank → top-3 | Wide net, precise final |
| Cache | query + result | ~80% hit rate typical |
| Embedding | batch of 100 | Efficiency |
| Refresh | daily incremental | Updates without full rebuild |

**Cost:** indexing one-time ~$1; per query ~$0.002; ~1K queries/day → ~$60/mo + vector DB (~$100-500/mo).
**Targets:** latency <1s P95, precision@3 >75%, cache hit >80%.

---

## 5.3 Design: 10K Concurrent Users

**Capacity math:**
```
10,000 concurrent users
~1 req/user/min steady   → 166 RPS
peak 5×                  → 830 RPS
burst 10×                → 1,660 RPS
P99 latency target       → <2s
```

**Layered architecture:**
```
TIER 1  API Gateway / LB  (geo-routing, rate limiting, TLS)
TIER 2  App Servers       (20-50 instances, async, auto-scale)
TIER 3  Cache (Redis cluster, 3× replication, ~80% hit)
TIER 4  Retrieval         (embedding workers + vector DB cluster)
TIER 5  LLM Queue         (smooths bursts) → Claude (multi-key) + fallback
TIER 6  Observability     (Prometheus metrics, Jaeger tracing, ELK logs)
```

**Component sizing:**
```
App servers: 1,660 RPS ÷ 100 RPS/server ≈ 17 → run 20 (auto-scale to 50)
Vector DB:   1,660 QPS ÷ 50 QPS/pod ≈ 34 pods (~$2.4K/mo)
Cache:       ~66 GB at 80% hit (~$400/mo)
LLM:         166 RPS × 500 tok ≈ 83K tok/s → ~$21.6K/mo
```
**Total ≈ $27K/mo ≈ $0.81 per 1K queries** (before optimization — see 5.6).

**Burst handling:** request queue with bounded depth + dynamic batching + circuit breaker + fallback to faster model (Haiku) when overloaded.

---

## 5.4 Design: 100K+ Documents

What changes as document count grows:

```
10K docs (50K chunks):    fits in memory / single vector index
100K docs (500K chunks):  needs distributed indexing (single node caps ~1M vectors)
1M+ docs:                 hierarchical search + quantization (IVF/HNSW)
```

**Search complexity:**
```
Brute force:   O(n)      ok up to ~100K vectors
Indexed (IVF): O(log n)  needed 100K-1M, build/maintenance cost
Hierarchical + quantization: 1M+ vectors
```

**Cost lever at scale — fine-tune embeddings:**
```
Generic embedding:    $0.02/1M tokens → ~$1 to index 500K chunks
Fine-tuned (domain):  $0.001/1M       → ~$0.05 (20× cheaper)
Trade-off: ~2-3% quality loss (usually acceptable); ROI ~2 months
```

---

## 5.5 Multi-Region Architecture

```
            GLOBAL LOAD BALANCER
            (route by geography / latency / health)
            ┌──────────────┴──────────────┐
       US-EAST (primary)             EU-WEST (secondary)
       app + cache + vectorDB        app + cache + vectorDB
            └────────── sync ─────────────┘

Sync strategy:
  Documents   master-master (5-60s lag, last-write-wins)
  Embeddings  eventual consistency (re-embed per region or nightly snapshot)
  Cache       independent per region (local optimization, no sync)
```

**Cost vs. reliability:**
```
1 region:  ~$27K/mo,  99%    uptime
2 regions: ~$54K/mo,  99.9%  uptime
3 regions: ~$81K/mo,  99.99% uptime
```

---

## 5.6 Cost Optimization at Scale

A worked example taking $27.1K/mo → $13.6K/mo (-50%) with <1% quality loss:

```
OPT 1  Fine-tune embeddings   $0.02→$0.001/1M    -$1.1K/mo  (-95% embed cost)
OPT 2  Aggressive caching     80%→90% hit rate   -$2.5K/mo  (-40% LLM cost)
OPT 3  Model routing          40% Haiku, 50% Sonnet, 10% Opus  -$10K/mo (-50%)
OPT 4  Batching/async         100× fewer embed calls          -$0.4K/mo
─────────────────────────────────────────────────────────────────────
TOTAL  -$13.5K/mo  →  $0.41 per 1K queries (from $0.81)
Quality: precision −1%, answer quality −1%, latency +50ms (imperceptible)
```

**De-risk:** gradual rollout (5% → 25% → 100%), close monitoring, A/B test each change, keep rollback path.

---

## 5.7 Hallucination Prevention

A defense-in-depth stack (cumulative effect from 50% → <1%):

```
LAYER 1  Retrieval quality (rerank + hybrid)        prevents ~30%
LAYER 2  Context quality (no contradictions, recent) prevents ~20%
LAYER 3  Prompt engineering ("ONLY use context")     prevents ~25%
LAYER 4  Confidence scoring (flag uncertain)         flags ~80% of rest
LAYER 5  Fact verification (claims vs sources)       prevents ~15%
LAYER 6  Human-in-loop (confidence < 60% → review)   eliminates remainder
```

Confidence score example: `0.3×retrieval_sim + 0.4×llm_confidence + 0.3×verification`.

---

## 5.8 Production Patterns

**Versioning** — keep document/embedding versions separate; deploy with canary (10% traffic) + atomic switch + rollback path. Use incremental re-embedding (only changed docs, ~20% typically) instead of full rebuilds.

**Caching hierarchy:**
```
L1 Query-result cache  (~20% hit)   saves entire pipeline
L2 Embedding cache     (~30% hit)   skips embedding API
L3 Vector-DB index     always       fast lookup
Eviction: LRU.  Sizing: in-memory(10GB) → Redis(50-100GB) → cluster(500GB+)
```

**Failover (circuit breaker pattern):**
```
Vector DB down  → retry w/ backoff → serve cached results → generic fallback
LLM rate-limited→ queue async → cheaper model (Haiku) → template answer
Cache down      → bypass, go direct (slower but available)
Circuit: if failures >50% for 10s → open (stop calling) → half-open test → close
```

---

## 5.9 Domain-Specific RAG

Each domain bends the architecture differently. **"Design RAG for a law firm" is a different answer than "for a startup"** — interviewers test whether you know this.

### Legal RAG
- **Constraints:** strict accuracy (errors = liability), precise citations (page/paragraph), compliance (privilege, GDPR), wording precision.
- **Adaptations:** smaller chunks (300 chars) for precision; mandatory reranking; store exact source location in metadata for citations; audit trails; human review for high-stakes.

### Medical RAG
- **Constraints:** hallucination = patient harm; evidence-based (peer-reviewed sources); HIPAA; diagnosis = liability.
- **Adaptations:** classify clinical vs. educational queries; for clinical, refuse diagnosis/treatment advice, require sources, always add "consult a provider"; selective retrieval (rerank to top-2); strict prompts.

### Financial RAG
- **Constraints:** SEC/FINRA compliance; market-moving disclosures; mandatory audit trails; numeric precision (off-by-one = fraud).
- **Adaptations:** explicit number/fact extraction step; flag material changes vs. prior statements; full audit log (who/what/when); compliance gate before release.

### Code RAG
- **Constraints:** syntax sensitivity (one char breaks it); multi-language; versioning (API changes); dependency compatibility.
- **Adaptations:** language-aware retrieval/filtering; preserve code blocks intact during chunking; surface version info; "always test before production" guardrail.

---
---

# PART 6 — INTERVIEW MASTERY

## 6.1 Question Bank by Level

> For each question: the concise answer, then the *depth signal* interviewers listen for.

### JUNIOR (0-2 years) — Fundamentals

**Q1. What is RAG and why do we need it?**
Three steps (retrieve → augment → generate) that ground the LLM in real data, solving knowledge-cutoff and proprietary-data gaps. *Depth:* explain it without jargon; note RAG informs rather than makes the model "smarter."

**Q2. Design a simple RAG system.**
Load → chunk (500/100) → embed (text-embedding-3-small) → store → on query: embed, retrieve top-3 (cosine), build prompt, call LLM. *Depth:* justify each default; mention how you'd handle stale docs.

**Q3. What are embeddings and why do they work?**
Vectors (~1,536-d) trained so similar meanings → similar vectors; compared via cosine similarity. *Depth:* contrastive training; semantic vs. lexical; why this beats keyword search.

**Q4. Compare embedding models.**
small (cheap/good), large (better/pricier), open-source (free/self-host), fine-tuned (best/domain). *Depth:* MVP→small; accuracy-critical→large; privacy→self-host; domain→fine-tune.

**Q5. What is chunking and why does it matter?**
Split docs to cut cost, reduce noise, improve retrieval. *Depth:* the precision-vs-context trade-off; when overlap matters; chunking a contract ≠ chunking a FAQ.

**Q6. Describe the retrieval process.**
Embed query → cosine vs. all chunks → rank → top-k. *Depth:* why top-3 not top-1; what to do when all similarities are low.

**Q7. What problems happen in RAG?**
Classify by source: data / retrieval / generation. *Depth:* the 4-layer diagnosis (Section 4.6).

### SENIOR (3-5 years) — Systems & Trade-offs

**Q8. Design RAG for 10K documents.** → Section 5.2. *Depth:* component sizing, cost breakdown, metrics/targets.

**Q9. Scale to 100K documents.** → Section 5.4. *Depth:* distributed indexing, search complexity, fine-tuned embeddings for cost.

**Q10. Measure and improve RAG quality.** → precision/recall + RAGAS (4.5) + iterative diagnose→fix→measure loop. *Depth:* labeled test set, statistical significance.

### LEAD/ARCHITECT (6+ years) — Complex Scenarios

**Q11. Design RAG for 10K concurrent users.** → Section 5.3. *Depth:* capacity math, burst handling, circuit breakers, observability.

**Q12. Multi-region RAG.** → Section 5.5. *Depth:* sync strategies, consistency trade-offs, cost vs. uptime.

**Q13. Cost optimization at scale.** → Section 5.6. *Depth:* quantified levers, de-risked rollout, quality monitoring.

**Q14. Handle hallucination.** → Section 5.7. *Depth:* layered defense with cumulative impact, confidence scoring, human-in-loop for high stakes.

### ADVANCED — Techniques & Domains

**Q15. Design RAG for legal documents.** → Section 5.9 Legal. *Depth:* citations, compliance, audit trails, human review.

**Q16. Handle multi-hop questions.** → Section 3.1. *Depth:* iterative retrieval, query expansion, knowledge graphs.

**Q17. Optimize for 200K context models.** → Section 3.5. *Depth:* shift from precision-retrieval to organization/synthesis.

**Q18. Diagnose a RAG failure.** → Section 4.6. *Depth:* 4-layer method, tools per layer, when RAG makes it worse.

**Q19. Build RAG for [your domain].** → Section 5.9. *Depth:* domain constraints drive architecture choices.

**Q20. Reduce LLM cost 50% without quality loss.** → Section 5.6 routing + caching + fine-tuned embeddings.

**Q21. RAG vs. fine-tuning — when each?** → Section 1.1. RAG=knowledge, fine-tuning=behavior; complementary.

**Q22. Why does reranking help if embeddings already rank?** → Section 1.6. Similarity ≠ relevance.

**Q23. Hybrid search — when and why?** → Section 2.6 / 4.3. Combines exact-match and semantic strengths.

**Q24. How do you evaluate without ground-truth labels?** → LLM-as-judge / RAGAS (4.5); bootstrap a labeled set from production + human spot-checks.

**Q25. What changed in RAG recently?** → Section 3.8. Long-context shifting strategy, multimodal, agentic RAG.

---

## 6.2 Common Interview Mistakes

| # | Mistake | Better Response |
|---|---------|-----------------|
| 1 | "Just increase top-k" | Diagnose root cause; more docs = more noise |
| 2 | Proposing without trade-offs | "Reranking: +10% accuracy, +200ms, +$0.001 — worth it if accuracy dominates" |
| 3 | Ignoring cost | Start with cheap model/embedding; upgrade only when metrics demand |
| 4 | No failure modes | Always state what breaks and the fallback |
| 5 | No monitoring plan | Name the metrics + alert thresholds |
| 6 | Ignoring cold start | Pre-warm cache, gradual traffic ramp |
| 7 | One-size-fits-all | Route by query complexity / domain |
| 8 | No systems thinking | "Optimizing for X, trading Y for Z; at 10× scale we'd need…" |

---

## 6.3 Interview Strategy Framework

Run every system-design answer through these seven moves, in order:

```
1. CLARIFY      What are we optimizing for? (scale, latency, cost, accuracy?)
2. HIGH-LEVEL   Sketch the architecture end to end
3. DEEP DIVE    Detail the 1-2 components that matter most
4. TRADE-OFFS   State what each choice costs explicitly
5. FAILURE      What breaks? How do we recover?
6. NUMBERS      Estimate latency, cost, capacity (don't be vague)
7. MEASURE      How would you know it works? Which metrics?
```

Self-score: completeness, depth (the *why*), trade-off awareness, failure handling, scale reasoning, intellectual humility.

---

## 6.4 Final Checklist

Before any RAG interview, confirm you can:

- [ ] Explain RAG to a non-technical person (Section 1)
- [ ] Build a working system from memory (Section 2.1)
- [ ] Run the 4-layer failure diagnosis (Section 4.6)
- [ ] Design for 10K docs with cost/metrics (Section 5.2)
- [ ] Design for 10K users with capacity math (Section 5.3)
- [ ] Cut cost 50% with named levers (Section 5.6)
- [ ] Explain embeddings, attention, IR theory (Part 4)
- [ ] Discuss trade-offs with numbers, not adjectives
- [ ] Adapt the design for legal/medical/financial (Section 5.9)
- [ ] Name what's new in RAG (Section 3.8)

---
---

# PART 7 — QUICK REFERENCE

## 7.1 API Cheat Sheet

```python
# EMBEDDINGS
from openai import OpenAI
client = OpenAI()
v = client.embeddings.create(input="text",
        model="text-embedding-3-small").data[0].embedding   # ~1,536-d

# CLAUDE MESSAGES
from anthropic import Anthropic
c = Anthropic()
ans = c.messages.create(
    model="claude-opus-4-20250805", max_tokens=1024,
    system="You are helpful",
    messages=[{"role": "user", "content": "..."}]
).content[0].text

# TOOL USE
resp = c.messages.create(
    model="claude-opus-4-20250805", max_tokens=1024,
    tools=[{"name": "...", "description": "...", "input_schema": {...}}],
    messages=[...])
for block in resp.content:
    if block.type == "tool_use":
        name, inp = block.name, block.input
```

## 7.2 Parameters & Cost Tables

```
PARAMETERS
chunk_size  500    start here; tune to data
overlap     100    preserve boundary context
top_k       3      retrieve;  10 if reranking down to 3
max_tokens  1024   response cap
temperature 0.3    LOW for RAG (factual)

MODELS (relative)
text-embedding-3-small   $0.02/1M   general/MVP
text-embedding-3-large   $0.13/1M   accuracy-critical
claude-opus-4            premium    complex reasoning
claude-3-5-haiku         ~3× cheaper simple/fast

COST PER QUERY
embed query   ~$0.0002
claude call   ~$0.0018
total         ~$0.002   (1K/day → ~$60/mo)
```

## 7.3 Troubleshooting Guide

| Symptom | Check | Fix |
|---------|-------|-----|
| "No relevant docs" | `len(rag.chunks)`; query sanity | raise top_k; check chunk_size; run `diagnose()` |
| Wrong/incomplete answer | what chunks retrieved? prompt clear? | enable reranking; hybrid search; tighten prompt |
| Slow | embedding? reranking? top_k? | cache embeddings; skip rerank for simple; lower top_k; Haiku |
| High cost | which component dominates? | fine-tune embeddings; cache; cheaper model; route |

## 7.4 Glossary

- **RAG** — Retrieval-Augmented Generation: ground an LLM in retrieved documents.
- **Embedding** — vector representation of text meaning (~1,536-d).
- **Cosine similarity** — angle between vectors; 1=same meaning, 0=unrelated.
- **Chunk** — a segment of a document, indexed and retrieved as a unit.
- **Top-k** — number of documents retrieved per query.
- **Reranking** — re-sorting retrieved candidates by *relevance* (not just similarity).
- **Hybrid search** — combining semantic + keyword retrieval.
- **Hallucination** — confident but false LLM output.
- **RAGAS** — RAG evaluation: faithfulness, answer/context relevance, utilization.
- **Multi-hop** — iterative retrieval across multiple documents.
- **BM25** — classic keyword-ranking algorithm (TF-IDF based).
- **Context window** — max tokens an LLM processes at once (Opus: 200K).
- **Temperature** — sampling randomness; low = factual, high = creative.
- **Circuit breaker** — pattern that stops calling a failing dependency.

---

*End of the Complete RAG Master Guide. Read it once to learn, keep it open to reference, review Part 6 before interviews.*
