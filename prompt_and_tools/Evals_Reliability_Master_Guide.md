# 📊 Evals & Reliability: Complete Master Guide
## Evaluation Frameworks, Metrics, Testing, and Production Quality Assurance

> **Level:** Senior-level understanding (suitable for interviews and production systems)  
> **Scope:** Everything about measuring, evaluating, and improving GenAI systems  
> **Philosophy:** You can't improve what you can't measure. Evals come first.

---

## 📋 Table of Contents

### Part 1: Why Evals Matter (The Mindset)
- [1.1: The Core Truth About Evals](#11-the-core-truth-about-evals)
- [1.2: The Three Levels of Quality](#12-the-three-levels-of-quality)
- [1.3: Evals vs Testing vs Monitoring](#13-evals-vs-testing-vs-monitoring)

### Part 2: Building Golden Datasets
- [2.1: What Is a Golden Dataset?](#21-what-is-a-golden-dataset)
- [2.2: Curating High-Quality Examples](#22-curating-high-quality-examples)
- [2.3: Dataset Challenges & Solutions](#23-dataset-challenges--solutions)
- [2.4: Versioning & Maintenance](#24-versioning--maintenance)

### Part 3: Metrics & Measurement
- [3.1: The Metrics Hierarchy](#31-the-metrics-hierarchy)
- [3.2: Retrieval Metrics (RAG)](#32-retrieval-metrics-rag)
- [3.3: Generation Metrics (Output Quality)](#33-generation-metrics-output-quality)
- [3.4: Agent & System Metrics](#34-agent--system-metrics)

### Part 4: Evaluation Frameworks & Tools
- [4.1: Building Your Own Evaluator](#41-building-your-own-evaluator)
- [4.2: RAGAs Framework](#42-ragas-framework)
- [4.3: DeepEval & Other Tools](#43-deepeval--other-tools)
- [4.4: LLM-as-Judge Pattern](#44-llm-as-judge-pattern)

### Part 5: Reliability Patterns
- [5.1: Detecting Hallucinations](#51-detecting-hallucinations)
- [5.2: Confidence Scoring](#52-confidence-scoring)
- [5.3: Fallback Strategies](#53-fallback-strategies)
- [5.4: Guardrails & Validation](#54-guardrails--validation)

### Part 6: Production Evals
- [6.1: Shadow Mode Evaluation](#61-shadow-mode-evaluation)
- [6.2: Regression Testing](#62-regression-testing)
- [6.3: Live Monitoring & Feedback](#63-live-monitoring--feedback)
- [6.4: Continuous Improvement](#64-continuous-improvement)

### Part 7: Interview Q&A
- [7.1: Core Questions](#71-core-questions)
- [7.2: Practical Questions](#72-practical-questions)
- [7.3: Production Questions](#73-production-questions)

---

# Part 1: Why Evals Matter (The Mindset)

## 1.1 The Core Truth About Evals

### The Uncomfortable Reality

Most teams spend:
- 80% time building features
- 15% time optimizing
- 5% time measuring

This should be inverted to:

- 40% time building
- 40% time measuring/evaluating
- 20% time optimizing based on metrics

**Why?** Because measurement enables everything else.

### The Measurement Equation

```
No Measurement:
"Does my RAG work?"
→ "Seems okay, the demo looks good"
→ Ship to production
→ Customer complains
→ "Why didn't it work?" (no data)

With Measurement:
"Does my RAG work?"
→ Test on 100 queries: Recall@5=0.68, Precision@5=0.72
→ Identify bottleneck: Chunking strategy losing context
→ Change chunking, re-test: Recall@5=0.81
→ Ship to production with confidence
→ "Here's why it works and how we know" (data-driven)
```

### What Evals Reveal

Evals are like X-rays for your system. They show:

```
Good metrics: ✓ System is healthy, ready to ship
Bad metrics:  ✗ System has problems (and exactly where)
Trends:       ↗ System improving or degrading over time
Gaps:         System works on easy cases, fails on hard ones
```

---

## 1.2 The Three Levels of Quality

### Level 1: Does It Work At All?

```
Binary question: Does the system produce ANY correct output?

Example RAG:
Input: "What is Paris?"
Output: "Paris is a city"
Check: Is this correct? Yes/No

Metric: Accuracy or Exact Match (0-100%)

You need: 50-100 examples minimum
Difficulty: Easy to measure
Usefulness: Foundation-level (you should start here)
```

### Level 2: How Well Does It Work?

```
Comparative question: How does it rank among alternatives?

Example RAG ranking:
Query: "Population of Paris?"
Option A: "2.1 million" (correct)
Option B: "Around 2 million" (approximately correct)
Option C: "France" (wrong)

Metric: Ranking of correctness (NDCG, MRR)

You need: 100-500 examples with ranked answers
Difficulty: Medium (requires good labels)
Usefulness: Competitive assessment (how much better is our system?)
```

### Level 3: When Does It Fail?

```
Diagnostic question: Under what conditions does it break?

Example RAG failure modes:
- Fails on domain-specific terminology
- Fails when answer requires multiple sources
- Fails on recent events (knowledge cutoff)
- Fails on contradictory information

Metric: Performance broken down by category/condition

You need: 500+ examples labeled with attributes
Difficulty: Hard (requires deep analysis)
Usefulness: Highest (tells you exactly what to fix)
```

### Progression

```
Beginner team:
"Does it work?" → Accuracy = 70%
(Better than guessing, but not actionable)

Intermediate team:
"How well?" → NDCG@10 = 0.82 vs competitor 0.75
(Now we know we're better, but why?)

Senior team:
"When does it fail?" → 85% on factual, 60% on reasoning, 40% on edge cases
(Now we know exactly where to invest engineering effort)
```

---

## 1.3 Evals vs Testing vs Monitoring

### Three Different Activities

| Activity | Question | Frequency | Output | Example |
|----------|----------|-----------|--------|---------|
| **Evals** | "How good is it?" | Before shipping (1-5% of time) | Metrics on golden dataset | "Recall@5=0.78 on 500 queries" |
| **Testing** | "Did I break it?" | During development (per commit) | Pass/fail on regression test | "14/15 tests passed" |
| **Monitoring** | "Is it still working?" | During production (continuous) | Health metrics in real-time | "Success rate=94%, avg latency=250ms" |

### Concrete Example: Shipping a Better RAG System

```
Timeline:

Day 1-3: Development
├─ Write code
├─ Run against 100 golden queries
├─ Eval result: Recall@5 improved from 0.68 → 0.75  ✓ EVALS
└─ Ready to test

Day 4: Testing Phase
├─ Run regression suite (20 critical queries)
├─ Check: Does new version still pass old tests?
├─ Result: 19/20 pass, 1 regression ✗ TESTING
└─ Fix the regression, re-test

Day 5: Deployment
├─ Monitor production health
├─ Track: % of successful retrievals
├─ Alert: If success rate drops below 90%
└─ Daily report: 98% success, serving 1000 queries/hr ✓ MONITORING
```

### The Relationship

```
           Golden Dataset
                 ↓
    ┌────────────Evals────────────┐
    │                             │
    ↓                             ↓
[Good Metrics]              [Bad Metrics]
    ↓                             ↓
  Build                       Iterate
    ↓                             ↓
    └──────→ Testing ←────────────┘
             (regression)
                 ↓
           [Passed Tests]
                 ↓
          Production Deploy
                 ↓
          Monitoring (live)
```

---

# Part 2: Building Golden Datasets

## 2.1 What Is a Golden Dataset?

### Definition

A **golden dataset** is a curated collection of input-output pairs that you trust as ground truth. It's your source of truth for measuring system quality.

```python
# Example golden dataset for RAG

golden_dataset = [
    {
        "query": "What is the population of Paris?",
        "expected_answer": "Approximately 2.1 million in the city proper",
        "relevant_documents": ["paris_demographics.pdf", "france_stats.pdf"],
        "quality": "high",  # How confident are we in this answer?
        "difficulty": "easy",  # Is this a hard or easy question?
        "source": "manual_curation",
        "tags": ["geography", "factual", "recent"]
    },
    {
        "query": "How does machine learning differ from deep learning?",
        "expected_answer": "Machine learning is a broader field...",
        "relevant_documents": ["ml_overview.pdf"],
        "quality": "high",
        "difficulty": "medium",
        "source": "expert_review",
        "tags": ["ai", "conceptual"]
    }
]
```

### Why It's Called "Golden"

```
"Golden" = precious, reliable, worth protecting

Not:
- "Perfect" (even experts disagree on some answers)
- "Comprehensive" (can't cover every possible query)
- "Static" (should evolve as the world changes)

But:
- Worth trusting (verified by humans)
- Represents diverse cases (easy, medium, hard)
- Maintainable (can be updated)
```

---

## 2.2 Curating High-Quality Examples

### The Curation Process

#### Step 1: Source Examples

Where do examples come from?

```python
sources = {
    "Production logs": {
        "description": "Real user queries from your system",
        "pros": "Realistic, represents actual usage",
        "cons": "Biased toward easy cases (hard cases fail silently)",
        "weight": "50% of dataset"
    },
    "Manual creation": {
        "description": "Experts manually write test cases",
        "pros": "Can target edge cases and hard scenarios",
        "cons": "Time-consuming, may not reflect real usage",
        "weight": "30% of dataset"
    },
    "Public benchmarks": {
        "description": "Existing datasets (SQuAD, MMLU, etc)",
        "pros": "Proven quality, comparable to others",
        "cons": "May not match your domain",
        "weight": "20% of dataset"
    }
}
```

#### Step 2: Label / Verify

For each example, you need the "correct" answer.

```python
def verify_answer(query: str, answer: str, source_docs: list) -> VerificationResult:
    """
    Human expert verifies: Is this answer correct given the documents?
    """
    # Could be:
    # 1. Exact: "Paris" vs "Paris" → definitely correct
    # 2. Equivalent: "2.1M" vs "2,100,000" → correct
    # 3. Partial: "2M" vs "2.1M" → sort of correct
    # 4. Wrong: "3M" vs "2.1M" → incorrect
    
    return {
        "is_correct": True/False,
        "confidence": 0.95,  # How sure are we?
        "reasoning": "Verified against official census data"
    }
```

#### Step 3: Rate Quality

Not all correct answers are equally valuable.

```python
class DataQuality(Enum):
    HIGH = "high"        # Verified by expert, documents are authoritative
    MEDIUM = "medium"    # Reasonable, but some uncertainty
    LOW = "low"          # Placeholder, needs review

# Example: Different quality levels
{
    "query": "What is the capital of France?",
    "answer": "Paris",
    "quality": "HIGH",  # Unambiguous, verified
    "confidence": 0.99
}

{
    "query": "Is quantum computing the future?",
    "answer": "Probably, for certain applications",
    "quality": "MEDIUM",  # Opinion-based, subjective
    "confidence": 0.50
}

{
    "query": "What's the latest stock price of AAPL?",
    "answer": "???",
    "quality": "LOW",  # Changes daily, not suitable for golden dataset
    "confidence": 0.10
}
```

#### Step 4: Annotate with Metadata

Add context so you can slice and analyze later.

```python
example = {
    "query": "How do I make a soufflé?",
    "answer": "A soufflé requires...",
    
    # Quality tracking
    "quality": "high",
    "verified_by": "chef_alice",
    "verified_date": "2025-03-01",
    
    # Categorization
    "category": "how-to",
    "domain": "cooking",
    "difficulty": "medium",  # Easy, medium, hard
    
    # Attributes (for error analysis)
    "requires_multi_step": True,
    "requires_domain_knowledge": True,
    "is_subjective": False,
    
    # Tracking
    "source": "production_log",
    "used_in_evals": True,
    "revision": 2
}
```

---

## 2.3 Dataset Challenges & Solutions

### Challenge 1: Ambiguous Correct Answers

```
Query: "Who is the best AI researcher?"
Multiple correct answers:
- Yann LeCun
- Yoshua Bengio
- Demis Hassabis
- ...

Problem: How do you eval against multiple correct answers?

Solution: Multiple-answer acceptance
```python
{
    "query": "Who is a leading AI researcher?",
    "acceptable_answers": [
        {"answer": "Yann LeCun", "confidence": 1.0},
        {"answer": "Yoshua Bengio", "confidence": 1.0},
        {"answer": "Demis Hassabis", "confidence": 1.0},
    ],
    "metric": "If output matches ANY acceptable answer → correct"
}
```

### Challenge 2: Outdated Information

```
Query: "Who is the president of the US?"
Answer (2020): "Joe Biden"
Answer (2024): Still "Joe Biden"
Answer (2028): TBD

Problem: Golden dataset becomes stale

Solution: Mark temporal data with expiration dates
```python
{
    "query": "Who is the US president?",
    "answer": "Joe Biden",
    "valid_from": "2021-01-20",
    "valid_until": "2025-01-20",  # Next election
    "note": "This example should be updated after 2025"
}
```

### Challenge 3: Subjective Quality

```
Query: "Write a poem about spring"
Model output: "Flowers bloom, birds sing, warm sun appears"

Problem: Is this a good poem? (subjective)

Solution: Attribute-based evaluation instead of binary
```python
{
    "query": "Write a poem about spring",
    "evaluation_rubric": {
        "contains_sensory_imagery": 5,  # out of 5
        "follows_meter": 2,
        "uses_metaphor": 4,
        "rhyme_scheme": 3
    },
    "overall_quality": 3.5,  # Average of attributes
    "note": "No single correct answer; evaluate on dimensions"
}
```

### Challenge 4: Cost of Curation

```
Realizing: Creating a good golden dataset is expensive

Scale:
- 100 examples: 5-10 hours (1-2 per minute, with verification)
- 500 examples: 30-50 hours
- 1000 examples: 60-100 hours

Solution: Start small, iterate
- Build 50-100 high-quality examples first
- Use them to evaluate and improve system
- Gradually expand as you identify problematic areas
```

---

## 2.4 Versioning & Maintenance

### Why Version Your Dataset?

```
Scenario 1: You discover an error in the golden dataset
"We labeled this answer as correct, but it's actually wrong"
→ You need to track: Who found it? When? What was the fix?

Scenario 2: You add new examples
→ You need to know: Is this v1.0 or v1.1?
→ Which results are comparable?

Scenario 3: You evaluate two model versions
Model A vs Model B: 
→ If they used different golden datasets, results aren't comparable
```

### Versioning Structure

```python
class GoldenDataset:
    def __init__(self):
        self.version = "1.0.0"  # Semantic versioning
        self.examples = []
        self.metadata = {
            "created_date": "2025-01-15",
            "last_updated": "2025-03-01",
            "created_by": "alice@company.com",
            "quality_level": "high",  # high, medium, low
            "coverage": {
                "easy_cases": 30,
                "medium_cases": 50,
                "hard_cases": 20
            }
        }
```

### Update Log

```python
update_log = [
    {
        "version": "1.0.0",
        "date": "2025-01-15",
        "changes": "Initial release, 100 examples",
        "author": "alice"
    },
    {
        "version": "1.0.1",
        "date": "2025-01-20",
        "changes": "Fixed incorrect label on example #42",
        "author": "bob",
        "examples_affected": [42]
    },
    {
        "version": "1.1.0",
        "date": "2025-02-01",
        "changes": "Added 50 hard edge case examples",
        "author": "alice",
        "new_examples": 50
    },
    {
        "version": "2.0.0",
        "date": "2025-03-01",
        "changes": "Updated to match new product domain",
        "author": "charlie",
        "examples_added": 150,
        "examples_removed": 50,
        "breaking_changes": True  # Results won't be comparable to v1.x
    }
]
```

---

# Part 3: Metrics & Measurement

## 3.1 The Metrics Hierarchy

### Start Simple, Add Complexity

```
Beginner Level:
✓ Exact Match (EM) — Output == expected output? Yes/No
✓ Accuracy — Percentage correct

Intermediate Level:
✓ Similarity metrics — How close is output to expected?
✓ Category-based — Different metrics for different types
✓ Human rater agreement — Do humans agree with the label?

Advanced Level:
✓ Task-specific metrics — NDCG for ranking, BLEU for translation
✓ Multi-dimensional — Measure different aspects
✓ Correlation analysis — Which metrics predict user satisfaction?
```

### The Metric Selection Flow

```
Question: "Is my system good?"
    ↓
Start with: Exact Match
Result: 70% (70% of answers match exactly)
Problem: Too strict (90% of close answers are good too)
    ↓
Try: Token overlap / semantic similarity
Result: 85% (more accurate)
Problem: Still doesn't capture partial credit
    ↓
Try: Multi-dimensional (reasoning, correctness, clarity)
Result: Reasoning=80%, Correctness=85%, Clarity=75%
Perfect: Now you know exactly what to improve
```

---

## 3.2 Retrieval Metrics (RAG)

### The RAG Evaluation Problem

```
For RAG systems, we care about TWO things:
1. Does the retriever find relevant documents?
2. Does the generator produce good answers given the docs?

Today: Focus on retrieval (Part 3.2)
Tomorrow: Focus on generation (Part 3.3)
```

### Recall@K

```
Question: "Of all the relevant documents, how many did we retrieve?"

Formula:
Recall@K = (relevant docs in top-K) / (total relevant docs)

Example:
Query: "Who won the 2024 Olympics gold in swimming?"
Relevant documents: doc_1, doc_2, doc_3 (3 total)
Top-5 retrieval results: doc_1, doc_5, doc_3, doc_2, doc_4
Recall@5 = 3/3 = 1.0  ✓ Found all relevant docs

Recall@3 = 2/3 = 0.67  ✗ Missed one in top-3

Interpretation:
- Recall@5 = 1.0 = "Good, we found everything by rank 5"
- Recall@1 = 0 = "Bad, first result was irrelevant"
- Recall@10 = 1.0 = "Found everything by rank 10"

Key insight: Higher K is easier (more chances to find docs)
Useful for: "Make sure we get relevant docs eventually"
```

### Precision@K

```
Question: "Of the documents we retrieved, how many were relevant?"

Formula:
Precision@K = (relevant docs in top-K) / K

Example (same as above):
Query: "Who won the 2024 Olympics gold in swimming?"
Relevant documents: doc_1, doc_2, doc_3
Top-5 retrieval: doc_1, doc_5, doc_3, doc_2, doc_4
Precision@5 = 3/5 = 0.6  ← 3 relevant out of 5 retrieved

Precision@3 = 2/3 = 0.67  ← 2 relevant out of 3 retrieved
Precision@1 = 1/1 = 1.0   ← first one was relevant

Interpretation:
- High precision = "Most results we return are useful"
- Low precision = "We return a lot of junk"

Useful for: "Avoid returning irrelevant results"
Trade-off: Recall wants to return everything, Precision wants to return only good stuff
```

### NDCG (Normalized Discounted Cumulative Gain)

```
Problem with Recall/Precision:
They don't care about RANKING
If doc_1 is most relevant, we want it at rank 1, not rank 5

Solution: NDCG accounts for ranking order

Formula (simplified):
NDCG@K = (relevance score / log(rank)) for each doc in top-K
          divided by (perfect ranking)

Example:
Query: "Machine learning tutorials"
Relevant docs: 
  - doc_A (most relevant, should be rank 1)
  - doc_B (medium relevant)
  - doc_C (least relevant)

Bad ranking (rank order):
1. doc_C (wrong order) → contributes little to NDCG
2. doc_A (should be first) → contributes medium
3. doc_B (okay) → contributes some

Good ranking:
1. doc_A (perfect!) → contributes a lot
2. doc_B (okay) → contributes some
3. doc_C (acceptable) → contributes little

NDCG@3 = 0.95 for good ranking vs 0.70 for bad ranking

Interpretation:
- NDCG = 1.0 = Perfect ranking (most relevant first)
- NDCG = 0.5 = Mediocre ranking (mixed order)
- NDCG = 0.2 = Poor ranking (relevant docs are buried)

Useful for: "Measure ranking quality holistically"
```

### MRR (Mean Reciprocal Rank)

```
Question: "How soon did we find the first relevant result?"

Formula:
MRR = 1 / (rank of first relevant doc)

Examples:
First relevant doc at rank 1: MRR = 1/1 = 1.0  ✓ Perfect
First relevant doc at rank 3: MRR = 1/3 = 0.33
First relevant doc at rank 5: MRR = 1/5 = 0.20
No relevant docs found:       MRR = 0.0

Interpretation:
- MRR = 1.0 = "Found relevant doc immediately"
- MRR = 0.5 = "Found relevant doc by rank 2"
- MRR = 0.0 = "Never found relevant doc"

Useful for: "Measure how quickly we find good results"
Trade-off: Cares only about the FIRST relevant doc, ignores quality after that
```

### Which Metric to Use?

```
Use Recall@K when: 
  → You want to capture all relevant documents
  → False negatives (missing docs) are costly
  → Example: "Detect all possible allergies in medical history"

Use Precision@K when:
  → You want to avoid junk results
  → False positives (wrong docs) are costly
  → Example: "Return only highly relevant documents"

Use NDCG@K when:
  → Ranking order matters (better docs should come first)
  → You want a balanced metric
  → Example: "Search engines need good ranking"

Use MRR when:
  → You just need to find ONE good result fast
  → Example: "Find the answer to a factual question"

Best practice: Use all four together
→ Get a complete picture of retrieval quality
```

---

## 3.3 Generation Metrics (Output Quality)

### Exact Match (EM)

```
Question: Does the model's answer exactly match the expected answer?

Implementation:
def exact_match(prediction: str, reference: str) -> bool:
    return prediction.strip().lower() == reference.strip().lower()

Example:
Expected: "Paris"
Prediction: "Paris" → EM = 1 (correct)
Prediction: "paris" → EM = 1 (correct, case-insensitive)
Prediction: "The city of Paris" → EM = 0 (wrong, extra words)

Pros:
✓ Simple to implement
✓ Unambiguous
✓ Fast to compute

Cons:
✗ Too strict (95% of close answers are wrong)
✗ Doesn't work for open-ended questions
✗ Discounts partial credit

When to use: Only for factual questions with single correct answer
```

### Token Overlap / F1 Score

```
Question: What percentage of tokens appear in both answers?

Implementation:
def token_overlap(prediction: str, reference: str) -> float:
    pred_tokens = set(prediction.lower().split())
    ref_tokens = set(reference.lower().split())
    overlap = pred_tokens & ref_tokens
    return len(overlap) / len(ref_tokens)  # Recall-style

def f1_score(prediction: str, reference: str) -> float:
    pred_tokens = set(prediction.lower().split())
    ref_tokens = set(reference.lower().split())
    overlap = len(pred_tokens & ref_tokens)
    
    precision = overlap / len(pred_tokens) if pred_tokens else 0
    recall = overlap / len(ref_tokens) if ref_tokens else 0
    
    if precision + recall == 0:
        return 0
    return 2 * (precision * recall) / (precision + recall)

Example:
Expected: "Paris is the capital of France"
Prediction: "The capital of France is Paris"
Tokens matching: {paris, is, the, capital, of, france} = 6/6
F1 = 1.0 (all tokens present, just reordered)

Expected: "Paris is the capital of France"
Prediction: "Paris is a beautiful city"
Tokens matching: {paris, is} = 2/6
F1 = 0.4 (some overlap, but missing key info)

Pros:
✓ More lenient than EM
✓ Fast to compute
✓ Accounts for synonymy somewhat

Cons:
✗ Word order doesn't matter (bad for nuanced answers)
✗ Penalizes paraphrasing (using different but correct words)
✗ Order of arguments doesn't matter (could get logic wrong)

When to use: Quick evaluation, needs more lenience than EM
```

### BLEU Score

```
Question: How similar is the predicted text to reference text?

Used in: Machine translation, summarization

Implementation:
from nltk.translate.bleu_score import sentence_bleu

def bleu(prediction: str, reference: str, weights=(0.25, 0.25, 0.25, 0.25)) -> float:
    reference_tokens = reference.split()
    prediction_tokens = prediction.split()
    return sentence_bleu([reference_tokens], prediction_tokens, weights=weights)

What it measures:
- 1-gram: single words (50% of score)
- 2-gram: word pairs (25%)
- 3-gram: word triplets (15%)
- 4-gram: word quads (10%)

Example:
Reference: "The cat sat on the mat"
Prediction: "The cat sat on the mat"
BLEU = 1.0 (perfect match)

Reference: "The cat sat on the mat"
Prediction: "The dog sat on the mat"
BLEU = 0.85 (mostly match, one word different)

Reference: "The cat sat on the mat"
Prediction: "Feline rested atop rug"
BLEU = 0.2 (different structure, similar meaning)

Pros:
✓ Accounts for word order
✓ Standardized, comparable across papers
✓ Correlates with human judgment reasonably well

Cons:
✗ Doesn't understand meaning (cat vs dog)
✗ Penalizes paraphrasing unfairly
✗ Requires exact token matches

When to use: Translation, summarization (needs strict format matching)
```

### ROUGE Score

```
Question: How much of the reference is covered by the prediction?

Used in: Summarization evaluation

Formula (simplified):
ROUGE-1: Recall of unigrams (single words)
ROUGE-L: Longest common subsequence

Example:
Reference: "The movie was great. Acting was superb."
Summary: "The movie was great. Acting was excellent."
ROUGE-1 = 7/10 = 0.7 (7 out of 10 words match)

Reference: "The movie was great. Acting was superb."
Summary: "The movie was great."
ROUGE-1 = 5/10 = 0.5 (captured main points)

Pros:
✓ Works well for summarization
✓ Accounts for partial matches
✓ Standard evaluation metric

Cons:
✗ Still doesn't understand meaning
✗ Penalizes synonyms (great vs excellent)

When to use: Summarization, headline generation
```

### Semantic Similarity (Embedding-based)

```
Question: Are the predicted and reference answers semantically equivalent?

Implementation:
from sentence_transformers import SentenceTransformer
import numpy as np

def semantic_similarity(prediction: str, reference: str) -> float:
    model = SentenceTransformer('all-MiniLM-L6-v2')
    
    pred_emb = model.encode(prediction)
    ref_emb = model.encode(reference)
    
    # Cosine similarity
    similarity = np.dot(pred_emb, ref_emb) / (
        np.linalg.norm(pred_emb) * np.linalg.norm(ref_emb)
    )
    return float(similarity)

Example:
Reference: "Paris is the capital of France"
Prediction: "France's capital city is Paris"
Semantic similarity = 0.98 (same meaning, different wording)

Reference: "Paris is the capital of France"
Prediction: "Paris is a beautiful city in Europe"
Semantic similarity = 0.85 (related but incomplete)

Reference: "Paris is the capital of France"
Prediction: "London is the capital of England"
Semantic similarity = 0.6 (similar structure, wrong content)

Pros:
✓ Understands synonymy (great vs excellent → similar)
✓ Handles paraphrasing well
✓ More human-like judgment

Cons:
✗ Slower to compute (embedding step)
✗ Depends on embedding model quality
✗ Can be overly lenient (ignores some errors)

When to use: Open-ended answers, semantic equivalence matters
```

### LLM-as-Judge (Evaluating with Another LLM)

```
Question: Have a language model evaluate another language model's output

Implementation:
def llm_judge(query: str, prediction: str, reference: str) -> float:
    prompt = f"""
    Query: {query}
    Expected Answer: {reference}
    Model Answer: {prediction}
    
    Rate the model answer on a scale of 1-5:
    1 = Completely wrong
    2 = Mostly wrong
    3 = Partially correct
    4 = Mostly correct
    5 = Completely correct
    
    Reasoning: [Explain your rating]
    Rating: [1-5]
    """
    
    rating = call_llm(prompt)  # Use GPT-4 or Claude
    return rating / 5.0  # Normalize to 0-1

Pros:
✓ Human-like reasoning
✓ Flexible, can handle complex criteria
✓ Understands nuance and context

Cons:
✗ Slow and expensive
✗ Non-deterministic (LLM output varies)
✗ Can have biases
✗ Hard to debug (what did the LLM actually judge?)

When to use: Final quality assessment, not during development
```

---

## 3.4 Agent & System Metrics

### Agent-Specific Metrics

```
When evaluating an AGENT (not just a single LLM call):

Completion rate: % of queries where agent reached final answer
- Goal: > 95%
- Problem: Agent loops infinitely

Iteration efficiency: Average iterations to complete
- Goal: < 5 iterations
- Problem: Agent making unnecessary tool calls

Tool call accuracy: % of correct tool selections
- Goal: > 90%
- Problem: Agent picking wrong tools

Final answer correctness: % of correct final answers
- Goal: > 85%
- Problem: Agent reasoning is off

Cost per query: $ spent per query
- Goal: Minimize while maintaining quality
- Problem: Trade-off between capability and cost
```

### System-Level Metrics

```
Latency:
- P50: Median time (50th percentile)
- P95: 95th percentile (accounts for slow cases)
- P99: 99th percentile (worst case)

Example:
P50 = 200ms (typical query takes 200ms)
P95 = 500ms (5% of queries take >500ms)
P99 = 2000ms (1% of queries take >2 seconds)

Throughput:
- Queries per second (QPS)
- Requests per minute (RPM)

Goal: Maximize throughput without exceeding latency SLA

Availability:
- % of time system is operational
- Example: 99.9% uptime = system down < 44 minutes/month

Error rate:
- % of requests that fail with errors
- Different from low-quality results (errors vs wrong answers)
```

---

# Part 4: Evaluation Frameworks & Tools

## 4.1 Building Your Own Evaluator

### Simple Custom Evaluator

```python
from typing import Callable
from dataclasses import dataclass

@dataclass
class EvaluationResult:
    metric_name: str
    score: float  # 0-1 or 0-100
    details: dict
    passing: bool  # Whether this meets threshold

class SimpleEvaluator:
    def __init__(self, threshold: float = 0.8):
        self.threshold = threshold
        self.results = []
    
    def add_metric(self, name: str, metric_fn: Callable):
        """Register a custom metric function."""
        self.metrics = getattr(self, 'metrics', {})
        self.metrics[name] = metric_fn
    
    def evaluate_example(self, 
                        query: str,
                        predicted_answer: str,
                        expected_answer: str,
                        context: str = "") -> dict:
        """Evaluate a single example."""
        
        results = {}
        
        # Exact match
        exact_match = self._exact_match(predicted_answer, expected_answer)
        results['exact_match'] = exact_match
        
        # Token overlap
        token_overlap = self._token_overlap(predicted_answer, expected_answer)
        results['token_overlap'] = token_overlap
        
        # Semantic similarity
        semantic_sim = self._semantic_similarity(predicted_answer, expected_answer)
        results['semantic_similarity'] = semantic_sim
        
        # Overall score
        overall = (exact_match + token_overlap + semantic_sim) / 3
        results['overall'] = overall
        
        # Did it pass?
        passed = overall >= self.threshold
        results['passed'] = passed
        
        return results
    
    def evaluate_dataset(self, examples: list) -> dict:
        """Evaluate multiple examples."""
        all_results = []
        
        for ex in examples:
            result = self.evaluate_example(
                ex['query'],
                ex['predicted_answer'],
                ex['expected_answer']
            )
            all_results.append(result)
        
        # Aggregate
        return {
            'average_score': sum(r['overall'] for r in all_results) / len(all_results),
            'pass_rate': sum(1 for r in all_results if r['passed']) / len(all_results),
            'by_metric': {
                'exact_match': sum(r['exact_match'] for r in all_results) / len(all_results),
                'token_overlap': sum(r['token_overlap'] for r in all_results) / len(all_results),
                'semantic_similarity': sum(r['semantic_similarity'] for r in all_results) / len(all_results)
            }
        }
    
    @staticmethod
    def _exact_match(pred: str, ref: str) -> float:
        return 1.0 if pred.strip().lower() == ref.strip().lower() else 0.0
    
    @staticmethod
    def _token_overlap(pred: str, ref: str) -> float:
        pred_tokens = set(pred.lower().split())
        ref_tokens = set(ref.lower().split())
        if not ref_tokens:
            return 1.0
        overlap = len(pred_tokens & ref_tokens)
        return overlap / len(ref_tokens)
    
    @staticmethod
    def _semantic_similarity(pred: str, ref: str) -> float:
        from sentence_transformers import SentenceTransformer
        import numpy as np
        
        model = SentenceTransformer('all-MiniLM-L6-v2')
        pred_emb = model.encode(pred)
        ref_emb = model.encode(ref)
        
        return float(np.dot(pred_emb, ref_emb) / (
            np.linalg.norm(pred_emb) * np.linalg.norm(ref_emb)
        ))

# Usage
evaluator = SimpleEvaluator(threshold=0.75)
results = evaluator.evaluate_dataset([
    {
        'query': 'What is Paris?',
        'predicted_answer': 'Paris is a city',
        'expected_answer': 'Paris is the capital of France'
    }
])
print(results)
```

---

## 4.2 RAGAs Framework

### What Is RAGAs?

RAGAs = **RAG Assessment System** — A framework specifically designed for evaluating RAG systems.

```python
# Installation
# pip install ragas

from ragas import evaluate
from ragas.metrics import (
    faithfulness,
    answer_relevancy,
    context_recall,
    context_precision,
)
from datasets import Dataset

# Create evaluation dataset
eval_dataset = Dataset.from_dict({
    'question': ['What is the capital of France?', '...'],
    'answer': ['Paris', '...'],
    'contexts': [['Paris is the capital...'], ...],
    'ground_truth': ['Paris', '...']
})

# Evaluate
results = evaluate(
    eval_dataset,
    metrics=[faithfulness, answer_relevancy, context_recall, context_precision]
)

print(results)
```

### Key RAGAs Metrics

| Metric | What It Measures | Range | Interpretation |
|--------|---|---|---|
| **Faithfulness** | Is answer grounded in context? | 0-1 | 0.9 = very grounded in docs |
| **Answer Relevancy** | Does answer address the question? | 0-1 | 0.85 = highly relevant |
| **Context Recall** | Did retriever get all relevant docs? | 0-1 | 0.8 = found 80% of relevant docs |
| **Context Precision** | What % of retrieved docs are useful? | 0-1 | 0.75 = 75% of retrieved docs helped |

### Complete RAGAs Example

```python
from ragas import evaluate
from ragas.metrics import (
    faithfulness,
    answer_relevancy,
    context_recall,
    context_precision,
)
from datasets import Dataset
from langchain.llms import OpenAI

# Prepare data (must include: question, answer, contexts, ground_truth)
data = {
    'question': [
        'What is machine learning?',
        'How does neural networks work?'
    ],
    'answer': [
        'ML is a subset of AI where systems learn from data',
        'Neural networks use interconnected layers to process data'
    ],
    'contexts': [
        [
            'Machine learning is a method of data analysis that automates analytical model building.',
            'It is a branch of artificial intelligence based on data'
        ],
        [
            'Neural networks are computing systems vaguely inspired by biological neural networks',
            'They consist of interconnected nodes or neurons'
        ]
    ],
    'ground_truth': [
        'Machine learning is a subset of artificial intelligence focused on data-driven learning',
        'Neural networks are models inspired by brain structure with interconnected layers'
    ]
}

eval_dataset = Dataset.from_dict(data)

# Evaluate (requires LLM API key for LLM-based metrics)
import os
os.environ["OPENAI_API_KEY"] = "your-key"

results = evaluate(
    eval_dataset,
    metrics=[
        faithfulness,
        answer_relevancy,
        context_recall,
        context_precision
    ]
)

# Results
print(f"Faithfulness: {results['faithfulness']:.3f}")
print(f"Answer Relevancy: {results['answer_relevancy']:.3f}")
print(f"Context Recall: {results['context_recall']:.3f}")
print(f"Context Precision: {results['context_precision']:.3f}")

# Detailed breakdown
for i, (q, a) in enumerate(zip(data['question'], data['answer'])):
    print(f"\nQuestion {i+1}: {q}")
    print(f"Faithfulness: {results['faithfulness'][i]:.3f}")
    print(f"Answer Relevancy: {results['answer_relevancy'][i]:.3f}")
```

---

## 4.3 DeepEval & Other Tools

### DeepEval

```python
# Installation
# pip install deepeval

from deepeval import evaluate
from deepeval.metrics import GEval, Faithfulness, AnswerRelevancy
from deepeval.test_case import LLMTestCase

# Create test cases
test_cases = [
    LLMTestCase(
        input="What is the capital of France?",
        actual_output="Paris",
        expected_output="Paris",
        retrieval_context=["Paris is the capital..."]
    ),
    # ... more test cases ...
]

# Define metrics
faithfulness = Faithfulness()
answer_relevancy = AnswerRelevancy()

# Custom metric using G-Eval
correctness = GEval(
    name="Correctness",
    criteria="Determine if the actual output is factually correct.",
    evaluation_steps=[
        "Check if the output addresses all parts of the input",
        "Verify factual accuracy against the retrieval context",
        "Assess completeness of the answer"
    ]
)

# Run evaluation
results = evaluate(
    test_cases,
    metrics=[faithfulness, answer_relevancy, correctness]
)
```

### Other Popular Tools

| Tool | What It Does | Best For |
|------|---|---|
| **RAGAs** | End-to-end RAG evaluation | RAG systems |
| **DeepEval** | Flexible metrics (G-Eval) | Custom evaluation logic |
| **LangSmith** | Production monitoring & debugging | Tracing tool use and agents |
| **Langfuse** | Open-source LangSmith alternative | Observability |
| **Weights & Biases** | ML experiment tracking | Comparing model versions |

---

## 4.4 LLM-as-Judge Pattern

### Why Use LLM-as-Judge?

```
Traditional metrics (exact match, BLEU):
- Fast, deterministic
- But too strict or too lenient

Human judges:
- Most accurate
- But expensive and slow

LLM-as-Judge:
- Fast (relative to humans)
- More nuanced than traditional metrics
- Can be standardized
- Still much cheaper than humans
```

### Implementation

```python
from anthropic import Anthropic

class LLMAsJudge:
    def __init__(self, model: str = "claude-opus-4-8"):
        self.client = Anthropic()
        self.model = model
    
    def judge(self,
             query: str,
             predicted_output: str,
             expected_output: str,
             rubric: str = None) -> dict:
        """Use LLM to judge quality of output."""
        
        rubric_str = rubric or """
        Rate the predicted output on these criteria:
        1. Correctness: Is the answer factually accurate?
        2. Completeness: Does it address all parts of the question?
        3. Clarity: Is the answer clear and well-structured?
        4. Relevance: Is all information relevant to the question?
        """
        
        prompt = f"""
        Query: {query}
        
        Expected/Reference Answer:
        {expected_output}
        
        Model's Predicted Answer:
        {predicted_output}
        
        {rubric_str}
        
        Please provide:
        1. A score (1-10) for each criterion
        2. An overall score (1-10)
        3. Brief reasoning for your judgment
        4. Specific strengths of the answer
        5. Specific areas for improvement
        
        Format your response as:
        CRITERION_SCORES:
        Correctness: X/10
        Completeness: X/10
        Clarity: X/10
        Relevance: X/10
        
        OVERALL_SCORE: X/10
        
        REASONING: [Your explanation]
        
        STRENGTHS: [List strengths]
        
        IMPROVEMENTS: [Suggest improvements]
        """
        
        response = self.client.messages.create(
            model=self.model,
            max_tokens=1024,
            messages=[{"role": "user", "content": prompt}]
        )
        
        # Parse response
        judgment_text = response.content[0].text
        overall_score = self._extract_overall_score(judgment_text)
        
        return {
            "raw_judgment": judgment_text,
            "overall_score": overall_score / 10.0,  # Normalize to 0-1
            "passed": overall_score >= 7  # Threshold
        }
    
    @staticmethod
    def _extract_overall_score(text: str) -> int:
        import re
        match = re.search(r'OVERALL_SCORE:\s*(\d+)', text)
        if match:
            return int(match.group(1))
        return 0

# Usage
judge = LLMAsJudge()

result = judge.judge(
    query="What is the capital of France?",
    predicted_output="Paris is the capital",
    expected_output="Paris"
)

print(f"Score: {result['overall_score']:.2f}")
print(f"Judgment: {result['raw_judgment']}")
```

---

# Part 5: Reliability Patterns

## 5.1 Detecting Hallucinations

### What Is a Hallucination?

```
Definition: When the model outputs confident-sounding but false information

Example:
Q: "What is the population of Paris?"
A: "The population of Paris is 5 million"  ← Hallucination (actually ~2.1M)

Key characteristic: Model is WRONG but SOUNDS RIGHT
This is worse than saying "I don't know"
```

### Detection Strategy 1: Grounding Check

```python
def grounding_check(answer: str, context: str) -> dict:
    """
    Does the answer exist in the provided context?
    (This is specifically for RAG systems)
    """
    
    # Extract key claims from answer
    claims = extract_claims(answer)
    
    grounded_claims = []
    hallucinated_claims = []
    
    for claim in claims:
        if claim_appears_in_context(claim, context):
            grounded_claims.append(claim)
        else:
            hallucinated_claims.append(claim)
    
    return {
        'grounded_percentage': len(grounded_claims) / len(claims),
        'grounded_claims': grounded_claims,
        'hallucinated_claims': hallucinated_claims,
        'is_hallucinating': len(hallucinated_claims) > 0
    }

# Example
answer = "Paris has 5 million people and is the capital of France"
context = "Paris is the capital of France with about 2.1 million residents"

result = grounding_check(answer, context)
print(result)
# Output:
# {
#     'grounded_percentage': 0.67,  # 2 out of 3 claims grounded
#     'hallucinated_claims': ['Paris has 5 million people']
# }
```

### Detection Strategy 2: Self-Contradiction Check

```python
def self_contradiction_check(answer: str) -> dict:
    """
    Does the model contradict itself within the answer?
    """
    
    # Check for contradictory statements
    prompt = f"""
    Check if this text contains any self-contradictions or logical inconsistencies:
    
    {answer}
    
    List any contradictions found. Format:
    CONTRADICTIONS:
    - [contradiction 1]
    - [contradiction 2]
    
    SEVERITY: High/Medium/Low
    
    EXPLANATION: [Why is this a problem?]
    """
    
    judgment = call_llm(prompt)
    
    has_contradictions = "CONTRADICTIONS:" in judgment and \
                        "None" not in judgment
    
    return {
        'has_contradictions': has_contradictions,
        'judgment': judgment
    }
```

### Detection Strategy 3: Confidence Calibration

```python
def calibration_check(answers: list) -> dict:
    """
    Does the model's confidence match its actual correctness?
    """
    
    results = []
    
    for answer in answers:
        # Extract confidence from answer
        confidence = extract_confidence_score(answer)  # 0-1
        
        # Check if actually correct
        is_correct = verify_correctness(answer)
        
        results.append({
            'predicted_confidence': confidence,
            'actual_correctness': 1 if is_correct else 0,
            'miscalibrated': confidence > 0.8 and not is_correct
        })
    
    calibration_error = mean([
        abs(r['predicted_confidence'] - r['actual_correctness'])
        for r in results
    ])
    
    return {
        'calibration_error': calibration_error,  # Lower = better
        'well_calibrated': calibration_error < 0.15,
        'results': results
    }
```

---

## 5.2 Confidence Scoring

### Why Confidence Matters

```
Model output: "The capital of France is Paris"
Without confidence: "Is this right? No way to know"

With confidence:
- "The capital of France is Paris" [confidence: 0.99]
- "The population of France is 5 million" [confidence: 0.45]

Now you can:
- Flag low-confidence outputs for human review
- Reduce hallucinations by not returning low-confidence answers
- Give users a sense of trust
```

### Implementation Patterns

#### Pattern 1: Token Probability

```python
def confidence_from_token_probability(response) -> float:
    """
    LLMs can expose the probability of each generated token.
    Higher average probability = more confident.
    """
    
    token_probs = response.log_probs  # Get from model
    
    # Average probability across tokens
    confidence = mean([exp(lp) for lp in token_probs])
    
    return confidence  # 0-1 scale

# Problem: Not all models expose token probabilities
# Alternative: Many models don't support this
```

#### Pattern 2: Explicit Confidence Prompt

```python
def confidence_from_prompt(query: str, answer: str) -> float:
    """
    Ask the model to rate its own confidence.
    """
    
    prompt = f"""
    Original question: {query}
    
    You just answered: {answer}
    
    On a scale of 0-100, how confident are you that this answer is:
    1. Factually accurate
    2. Complete (addresses the full question)
    3. Clear and well-explained
    
    Provide an overall confidence score (0-100):
    CONFIDENCE: [score]
    
    Reasoning: [Why this level of confidence?]
    """
    
    response = call_llm(prompt)
    score = extract_score(response)
    
    return score / 100.0  # Normalize to 0-1
```

#### Pattern 3: Ensemble Confidence

```python
def confidence_from_ensemble(query: str, num_samples: int = 5) -> dict:
    """
    If the model gives the same answer multiple times → high confidence.
    If it varies → low confidence.
    """
    
    answers = []
    for _ in range(num_samples):
        answer = call_model(
            query,
            temperature=0.7  # Non-zero for diversity
        )
        answers.append(answer)
    
    # Count agreement
    most_common = max(set(answers), key=answers.count)
    agreement_rate = answers.count(most_common) / num_samples
    
    return {
        'confidence': agreement_rate,  # 0.2-1.0
        'primary_answer': most_common,
        'all_answers': answers,
        'consensus': agreement_rate > 0.8
    }
```

---

## 5.3 Fallback Strategies

### When Confidence Is Low

```
Scenario: Model outputs answer with confidence 0.3

Options:

1. DON'T RETURN IT
   Instead: "I'm not sure. Would you like me to..."

2. FLAG FOR HUMAN REVIEW
   Instead: Send to human reviewer, return placeholder

3. TRY ALTERNATIVE APPROACH
   Instead: Use different method (RAG, search, etc.)

4. REQUEST CLARIFICATION
   Instead: "Could you rephrase the question?"
```

### Implementation

```python
def intelligent_fallback(query: str, fallback_chain: list) -> dict:
    """
    Try multiple strategies in order of preference.
    """
    
    for strategy in fallback_chain:
        result = strategy(query)
        
        if result['success'] and result['confidence'] > 0.7:
            return result
        
        # This strategy didn't work well, try next
    
    # All strategies failed
    return {
        'success': False,
        'answer': "I couldn't answer this question. " + \
                 "Please try rephrasing or ask a different question.",
        'reason': 'All fallback strategies failed'
    }

# Define fallback chain (in order of preference)
fallback_strategies = [
    primary_model,        # Best but maybe slow
    cached_answer,        # If we've seen this before
    retrieval_augmented,  # Search for context
    simple_baseline,      # Fast but less capable
]

result = intelligent_fallback(query, fallback_strategies)
```

---

## 5.4 Guardrails & Validation

### Input Validation

```python
def validate_input(query: str) -> tuple[bool, str]:
    """
    Check if input is safe and valid to process.
    """
    
    # Length check
    if len(query) > 10000:
        return False, "Query too long (max 10000 chars)"
    
    # Empty check
    if not query.strip():
        return False, "Query is empty"
    
    # Prohibited content check
    prohibited_patterns = [
        r"drop table",  # SQL injection
        r"<script>",    # XSS
        r"system\(",    # Command injection
    ]
    
    for pattern in prohibited_patterns:
        if re.search(pattern, query, re.IGNORECASE):
            return False, f"Query contains prohibited pattern: {pattern}"
    
    return True, ""

# Usage
is_valid, error = validate_input(user_query)
if not is_valid:
    return error_response(error)
```

### Output Validation

```python
def validate_output(answer: str, criteria: dict) -> tuple[bool, str]:
    """
    Check if output meets quality criteria.
    """
    
    # Length validation
    if len(answer) < criteria.get('min_length', 10):
        return False, "Answer too short"
    
    if len(answer) > criteria.get('max_length', 5000):
        return False, "Answer too long"
    
    # Format validation
    if criteria.get('required_format') == 'json':
        try:
            json.loads(answer)
        except:
            return False, "Output is not valid JSON"
    
    # Content validation
    if criteria.get('must_contain'):
        for required_phrase in criteria['must_contain']:
            if required_phrase not in answer:
                return False, f"Output missing required content: {required_phrase}"
    
    # Forbidden content check
    if criteria.get('must_not_contain'):
        for forbidden in criteria['must_not_contain']:
            if forbidden in answer:
                return False, f"Output contains prohibited content"
    
    return True, ""

# Usage
is_valid, error = validate_output(
    answer,
    criteria={
        'min_length': 50,
        'max_length': 2000,
        'required_format': 'text',
        'must_contain': ['evidence', 'reasoning'],
        'must_not_contain': ['I don\'t know', 'not sure']
    }
)

if not is_valid:
    return fallback_answer + f" (Note: {error})"
```

---

# Part 6: Production Evals

## 6.1 Shadow Mode Evaluation

### What Is Shadow Mode?

```
Normal deployment:
User → Model A → Response → User sees it

Shadow mode:
User → Model A → Response → User sees A
     ↓
     Model B (new version) → Response → We see it (hidden)

Benefit: Compare A vs B without affecting users
```

### Implementation

```python
def shadow_mode_comparison(user_query: str) -> dict:
    """
    Run both old and new model, compare without showing new version.
    """
    
    # Run current (production) model
    current_response = current_model(user_query)
    current_latency = measure_latency(current_response)
    
    # Run new (shadow) model
    new_response = new_model(user_query)
    new_latency = measure_latency(new_response)
    
    # Compare (but only show current response to user)
    comparison = {
        'user_query': user_query,
        'current_response': current_response,
        'new_response': new_response,
        'new_response_hidden_from_user': True,  # Important!
        
        'metrics': {
            'current_quality': evaluate_quality(current_response),
            'new_quality': evaluate_quality(new_response),
            'quality_improvement': evaluate_quality(new_response) - evaluate_quality(current_response),
            
            'current_latency_ms': current_latency,
            'new_latency_ms': new_latency,
            'latency_change_ms': new_latency - current_latency,
        }
    }
    
    # Log for analysis
    log_shadow_comparison(comparison)
    
    # Return ONLY current response to user
    return current_response
```

### Analysis After Shadow Mode

```python
def analyze_shadow_results(shadow_logs: list) -> dict:
    """
    After running shadow mode for N days, analyze results.
    """
    
    quality_improvements = [
        log['metrics']['quality_improvement']
        for log in shadow_logs
    ]
    
    latency_changes = [
        log['metrics']['latency_change_ms']
        for log in shadow_logs
    ]
    
    return {
        'avg_quality_improvement': mean(quality_improvements),
        'p95_quality_improvement': percentile(quality_improvements, 95),
        'avg_latency_change_ms': mean(latency_changes),
        'p95_latency_change_ms': percentile(latency_changes, 95),
        
        'recommendation': (
            "Deploy new version" if mean(quality_improvements) > 0.05 \
            else "Keep current version"
        )
    }
```

---

## 6.2 Regression Testing

### What Is Regression?

```
Regression = New version performs WORSE than old version on some cases

Example:
Model v1: "What is Paris?" → "Paris is the capital of France"
Model v2: "What is Paris?" → "Paris is a city"  ← Worse!

This is a REGRESSION. We broke something.
```

### Regression Test Suite

```python
class RegressionTestSuite:
    def __init__(self, critical_queries: list):
        self.critical_queries = critical_queries  # Most important cases
        self.baseline_results = {}  # v1 results
    
    def set_baseline(self, model):
        """Establish baseline from current model."""
        self.baseline_results = {}
        for query in self.critical_queries:
            response = model(query)
            score = evaluate_quality(response)
            self.baseline_results[query] = score
    
    def check_regressions(self, model, threshold: float = -0.05) -> dict:
        """
        Check new model against baseline.
        threshold = acceptable quality decrease (e.g., -5%)
        """
        
        regressions = []
        improvements = []
        no_change = []
        
        for query in self.critical_queries:
            new_response = model(query)
            new_score = evaluate_quality(new_response)
            baseline_score = self.baseline_results[query]
            
            delta = new_score - baseline_score
            
            if delta < threshold:
                regressions.append({
                    'query': query,
                    'baseline_score': baseline_score,
                    'new_score': new_score,
                    'delta': delta
                })
            elif delta > 0.05:
                improvements.append({
                    'query': query,
                    'delta': delta
                })
            else:
                no_change.append(query)
        
        return {
            'passed': len(regressions) == 0,
            'regressions': regressions,
            'improvements': improvements,
            'no_change_count': len(no_change)
        }

# Usage
suite = RegressionTestSuite(critical_queries=[
    "What is the capital of France?",
    "How do I make pizza?",
    # ... other critical queries
])

# Set baseline from current model
suite.set_baseline(current_model)

# Test new model
results = suite.check_regressions(new_model, threshold=-0.05)

if not results['passed']:
    print("FAILED REGRESSION TEST!")
    for reg in results['regressions']:
        print(f"  {reg['query']}: {reg['delta']:.3f}")
    # Don't deploy!
else:
    print("Passed regression test. Ready to deploy.")
```

---

## 6.3 Live Monitoring & Feedback

### Real-Time Metrics

```python
class ProductionMonitor:
    def __init__(self):
        self.metrics = {
            'success_rate': 0.0,
            'avg_latency_ms': 0.0,
            'error_rate': 0.0,
            'low_confidence_rate': 0.0,
        }
        self.recent_samples = []
    
    def record_request(self, query: str, response: dict, latency_ms: float):
        """Record a single user request."""
        
        sample = {
            'timestamp': datetime.now(),
            'query': query,
            'response': response,
            'latency_ms': latency_ms,
            'success': response.get('success', False),
            'confidence': response.get('confidence', 0.5),
            'error': response.get('error')
        }
        
        self.recent_samples.append(sample)
        
        # Keep only last 1000 samples for efficiency
        if len(self.recent_samples) > 1000:
            self.recent_samples.pop(0)
        
        # Update aggregate metrics
        self._update_metrics()
    
    def _update_metrics(self):
        """Recalculate metrics from recent samples."""
        
        if not self.recent_samples:
            return
        
        successes = sum(1 for s in self.recent_samples if s['success'])
        self.metrics['success_rate'] = successes / len(self.recent_samples)
        
        latencies = [s['latency_ms'] for s in self.recent_samples]
        self.metrics['avg_latency_ms'] = mean(latencies)
        
        errors = sum(1 for s in self.recent_samples if s['error'])
        self.metrics['error_rate'] = errors / len(self.recent_samples)
        
        low_conf = sum(1 for s in self.recent_samples if s['confidence'] < 0.6)
        self.metrics['low_confidence_rate'] = low_conf / len(self.recent_samples)
    
    def get_health_status(self) -> str:
        """Summarize health of system."""
        
        if self.metrics['error_rate'] > 0.05:
            return "CRITICAL: High error rate"
        if self.metrics['success_rate'] < 0.85:
            return "WARNING: Low success rate"
        if self.metrics['avg_latency_ms'] > 5000:
            return "WARNING: High latency"
        if self.metrics['low_confidence_rate'] > 0.3:
            return "WARNING: High uncertainty"
        
        return "HEALTHY"

# Usage
monitor = ProductionMonitor()

# In request handler
def handle_request(user_query: str):
    start = time.time()
    
    response = model(user_query)
    
    latency = (time.time() - start) * 1000
    monitor.record_request(user_query, response, latency)
    
    # Alert if needed
    if monitor.get_health_status() != "HEALTHY":
        send_alert(monitor.get_health_status())
    
    return response
```

### Explicit User Feedback

```python
class FeedbackCollector:
    def __init__(self):
        self.feedback_samples = []
    
    def collect_feedback(self, query: str, response: str, user_rating: int):
        """
        1 = Terrible
        2 = Bad
        3 = OK
        4 = Good
        5 = Excellent
        """
        
        self.feedback_samples.append({
            'timestamp': datetime.now(),
            'query': query,
            'response': response,
            'user_rating': user_rating
        })
    
    def get_satisfaction_metrics(self) -> dict:
        """Analyze user satisfaction."""
        
        if not self.feedback_samples:
            return {}
        
        ratings = [s['user_rating'] for s in self.feedback_samples]
        
        return {
            'avg_rating': mean(ratings),
            'satisfaction_rate': sum(1 for r in ratings if r >= 4) / len(ratings),
            'dissatisfaction_rate': sum(1 for r in ratings if r <= 2) / len(ratings),
            'total_feedback': len(ratings)
        }

# In UI
def show_response(response: str, query: str):
    """Show response and collect feedback."""
    
    print(response)
    
    # Ask user
    rating = input("How helpful was this? (1-5): ")
    
    feedback_collector.collect_feedback(query, response, int(rating))
    
    # Alert if dissatisfaction spike
    metrics = feedback_collector.get_satisfaction_metrics()
    if metrics['dissatisfaction_rate'] > 0.2:
        send_alert("High dissatisfaction detected!")
```

---

## 6.4 Continuous Improvement

### The Feedback Loop

```
User Query
    ↓
System Response
    ↓
User Feedback (1-5 stars)
    ↓
Identify failures (1-2 star responses)
    ↓
Add to failure analysis dataset
    ↓
Understand root cause
    ↓
Fix (better prompt, better tools, new training)
    ↓
Re-evaluate on golden dataset
    ↓
Deploy new version
    ↓
Monitor for regressions
```

### Implementation

```python
class ContinuousImprovementEngine:
    def __init__(self, golden_dataset: list):
        self.golden_dataset = golden_dataset
        self.failure_cases = []  # Cases where system failed
    
    def add_failure_case(self, query: str, response: str, user_rating: int):
        """User rated response as poor."""
        
        if user_rating <= 2:
            self.failure_cases.append({
                'query': query,
                'bad_response': response,
                'timestamp': datetime.now(),
                'severity': 'high' if user_rating == 1 else 'medium'
            })
    
    def analyze_failures(self) -> dict:
        """Understand why system is failing."""
        
        failure_patterns = {}
        
        for failure in self.failure_cases:
            # Categorize failure
            category = self._categorize_failure(failure['query'])
            
            if category not in failure_patterns:
                failure_patterns[category] = []
            failure_patterns[category].append(failure)
        
        return {
            'top_failure_categories': sorted(
                failure_patterns.items(),
                key=lambda x: len(x[1]),
                reverse=True
            )[:5],  # Top 5
            'total_failures': len(self.failure_cases),
            'failure_distribution': {
                cat: len(cases) for cat, cases in failure_patterns.items()
            }
        }
    
    def prioritize_improvements(self) -> list:
        """What should we fix first?"""
        
        analysis = self.analyze_failures()
        
        priorities = []
        for category, cases in analysis['top_failure_categories']:
            impact = len(cases)  # How many users affected
            priority = impact / 100  # Normalize
            
            priorities.append({
                'category': category,
                'affected_users': impact,
                'priority_score': priority,
                'example_cases': cases[:3]
            })
        
        return sorted(
            priorities,
            key=lambda x: x['priority_score'],
            reverse=True
        )
    
    def _categorize_failure(self, query: str) -> str:
        """Categorize what type of query failed."""
        
        if "how to" in query.lower():
            return "how-to-questions"
        elif any(symbol in query for symbol in ["?", "what", "when", "where"]):
            return "factual-questions"
        else:
            return "other"

# Usage
engine = ContinuousImprovementEngine(golden_dataset)

# Collect failures over time
for user_feedback in incoming_feedback:
    if user_feedback['rating'] <= 2:
        engine.add_failure_case(
            user_feedback['query'],
            user_feedback['response'],
            user_feedback['rating']
        )

# Weekly analysis
priorities = engine.prioritize_improvements()
print("Top issues to fix:")
for p in priorities[:3]:
    print(f"  {p['category']}: {p['affected_users']} users")
```

---

# Part 7: Interview Q&A

## 7.1 Core Questions

### Q1: Why Are Evals More Important Than Building?

**Strong Answer:**
> Measurement enables everything. Without evals, you don't know if you're improving or getting worse. I typically spend 40-50% of engineering time on evals because:
>
> 1. **Evals catch regressions early** — Prevents shipping broken code
> 2. **Evals guide prioritization** — Show exactly where to invest effort
> 3. **Evals enable debugging** — When something fails, metrics show why
> 4. **Evals build confidence** — You know the system actually works
>
> The math: 1 hour on evals saves 10 hours of debugging in production.

---

### Q2: What Metrics Would You Track for a RAG System?

**Strong Answer:**
> I'd measure both retrieval and generation quality:
>
> **Retrieval metrics:**
> - Recall@5: Did we retrieve all relevant docs? (Goal: >0.8)
> - Precision@5: What % of retrieved docs were useful? (Goal: >0.75)
> - NDCG@10: Is ranking good? (Goal: >0.75)
> 
> **Generation metrics:**
> - Faithfulness: Is answer grounded in retrieved docs? (Goal: >0.85)
> - Answer relevancy: Does it address the query? (Goal: >0.80)
> - Semantic similarity: How close to expected answer? (Goal: >0.85)
>
> **System metrics:**
> - Latency P95: < 2 seconds
> - Success rate: > 95%
> - Cost per query: Track for ROI
>
> I'd track these by category (easy vs hard queries) to identify weak areas.

---

### Q3: You Get Mediocre Eval Scores. Where Do You Start?

**Strong Answer:**
> Diagnosis first, solution second. I'd investigate:
>
> 1. **Golden dataset quality** — Are the expected answers actually correct? (Most common issue)
> 2. **Metric appropriateness** — Am I using the right metric? (E.g., exact match too strict)
> 3. **Breakdown by difficulty** — Is it failing only on hard cases? (Different fix than overall poor)
> 4. **Retrieval vs generation** — Which component is failing? (Fixes different things)
>
> Then based on the issue:
> - Bad dataset: Recurate it
> - Bad metric: Change how we measure
> - Weak retrieval: Improve chunking, embedding model, or ranking
> - Weak generation: Better prompt, few-shot examples, or fine-tuning

---

## 7.2 Practical Questions

### Q4: How Would You Build a Golden Dataset from Scratch?

**Strong Answer:**
> Iterative approach:
>
> **Week 1: Bootstrap (50 examples)**
> - Take 50 production queries where system worked well
> - Have expert verify answers
> - All "high quality" tier
>
> **Week 2-3: Expand (100 examples)**
> - Add 50 edge cases (things system struggled on)
> - Add questions about recent info (tests knowledge cutoff)
> - Mix: 60% high quality, 30% medium, 10% low
>
> **Week 4+: Maintain**
> - Add 10-20 new examples per week from production failures
> - Fix mislabeled examples as discovered
> - Version control (v1.0 → v1.1 → v1.2)
>
> **Key: Start small, iterate fast.** Don't try to build 1000 examples upfront (waste of time). Build 100 good ones and expand based on failures.

---

### Q5: A New Model Version Looks Better in Evals But Worse in Production. Why?

**Strong Answer:**
> Classic eval/prod mismatch. Possible causes:
>
> 1. **Golden dataset doesn't represent production** — Test queries differ from real user queries. Fix: Add real production queries to golden set.
>
> 2. **Metric doesn't correlate with satisfaction** — BLEU score improved but users hate it. Fix: Add human raters to validate.
>
> 3. **Edge cases not in evals** — System fails on rare cases. Fix: Expand golden dataset coverage.
>
> 4. **Model behaves differently at scale** — Works in test environment, fails with real load. Fix: Load test before deploying.
>
> **How I'd debug:**
> - Sample 100 users who rated responses
> - Compare old vs new on those real ratings
> - Find queries where new model is worse on human judgment
> - Understand why (usually dataset mismatch)

---

### Q6: Design an Eval Suite for a Customer Support Agent

**Strong Answer:**
> Three-tier system:
>
> **Tier 1: Correctness (40%)**
> - Is the answer technically correct?
> - Metric: Human raters score 1-5
> - Golden dataset: 200 customer queries + expert answers
>
> **Tier 2: Appropriateness (30%)**
> - Is the tone respectful? Grammar correct? Within policy?
> - Metric: LLM-as-judge checks policy compliance
> - Golden dataset: Examples of good vs bad responses
>
> **Tier 3: Resolution (30%)**
> - Did the agent actually solve the customer's problem?
> - Metric: Does customer have to follow up? (from logs)
> - Golden dataset: Real resolved and unresolved tickets
>
> **Monitoring in production:**
> - Track customer satisfaction (CSAT survey)
> - Track escalation rate (% escalated to humans)
> - Track resolution rate (% don't follow up)
> - Alert if CSAT drops by >5%

---

## 7.3 Production Questions

### Q7: Your System Has High Eval Scores But Hallucinating Often. What Went Wrong?

**Strong Answer:**
> Hallucinations are tricky — the system sounds confident when wrong. The issue is likely:
>
> 1. **Golden dataset doesn't test hallucination** — Evals only check "correct answers," not "wrong answers that sound right." Fix: Add adversarial examples (questions where obvious wrong answers sound plausible).
>
> 2. **Confidence metric is missing** — Model outputs high score even when uncertain. Fix: Add confidence scoring and flag low-confidence outputs.
>
> 3. **No grounding check** — For RAG, aren't checking if answer is grounded in retrieved docs. Fix: Add faithfulness metric.
>
> **Concrete fix:**
> ```python
> # Add to evals
> hallucination_test_cases = [
>     {
>         "query": "What is the capital of Atlantis?",
>         "golden_answer": "Atlantis is fictional; it doesn't have a capital",
>         "common_hallucination": "The capital of Atlantis is Poseidon's Palace"
>     }
> ]
> ```

---

### Q8: You Need to Ship Code Tomorrow. Your Evals Need 2 Weeks. What Do You Do?

**Strong Answer:**
> Don't skip evals, but be strategic:
>
> **Fast-track approach:**
> 1. **Critical cases only** — 30 essential queries (not 500)
> 2. **Exact match metric** — Fastest to evaluate (no LLM calls)
> 3. **Compare against baseline** — Run new vs old on these 30 cases
> 4. **Manual spot-check** — Have 2 people review 10 examples carefully
> 5. **Deploy with monitoring** — Watch production metrics closely
>
> **What I DON'T skip:**
> - Regression test on critical queries
> - Spot-check by domain expert
> - Production monitoring (alert thresholds)
>
> **What I DO defer:**
> - Full golden dataset evaluation (do it post-launch)
> - Confidence calibration (add later)
> - Comprehensive failure analysis (comes after more data)
>
> **Key principle:** Do 20% of evals work, catch 80% of problems. Ship. Polish evals based on production feedback.

---

### Q9: Monthly Cost is 40% From Eval LLM Calls. How Do You Optimize?

**Strong Answer:**
> Evals are expensive because they call models repeatedly. Optimization strategies:
>
> **Tier 1: Use Cheap Models (50% savings)**
> - Use Haiku for evaluation (cheaper than Opus)
> - Use cached golden dataset (don't re-evaluate everything)
> - Only use Opus for disputed cases
>
> **Tier 2: Batch Evaluation (30% savings)**
> - Evaluate 10 examples per API call instead of 1
> - Run evals on schedule (hourly, not per-request)
> - Cache results
>
> **Tier 3: Non-LLM Metrics (60% savings)**
> - Don't use LLM-as-judge for everything
> - Use cheaper metrics first (token overlap, semantic similarity)
> - Only use LLM judgment for ambiguous cases
>
> **Example reduction:**
> Before: $500/month (evaluating 1000 examples × 10 LLM calls each)
> After: $150/month (cheaper model + batching + non-LLM metrics)

---

### Q10: Design a Regression Test Suite for a Critical Production System

**Strong Answer:**
> Multi-layer approach:
>
> **Layer 1: Smoke Tests (5 minutes, before deploy)**
> - 10 critical queries: "Does system still work at all?"
> - Metric: Binary pass/fail
> - Threshold: 10/10 must pass
>
> **Layer 2: Regression Suite (1 hour, before deploy)**
> - 100 golden queries covering all features
> - Metric: Compare against baseline (allow <5% degradation)
> - Threshold: 95/100 must be acceptable
> - Breakdown by category (easy vs hard, domain-specific)
>
> **Layer 3: Canary Deployment (production, realtime)**
> - Serve 5% of traffic to new version
> - Monitor: P95 latency, error rate, low-confidence rate
> - Auto-rollback if any metric degrades >10%
>
> **Layer 4: Shadow Mode (production, 1 day)**
> - Run new version on 100% of traffic but hidden
> - Compare quality metrics vs current
> - Analyze failures before full rollout
>
> **Layer 5: Manual Spot-Check**
> - Domain expert reviews 20 critical examples
> - Catches things automation misses
>
> **Timeline:**
> - Layers 1-2: before deploy (1.5 hours)
> - Layers 3-4: during/after deploy (1 day)
> - Layer 5: before full rollout (30 min)

---

## Summary: Key Takeaways

### Core Principles
- ✅ **Measure first, optimize second** — You can't improve what you can't measure
- ✅ **Golden datasets are investments** — Build them iteratively, start small
- ✅ **Multiple metrics, not one** — Different metrics reveal different problems
- ✅ **Production evals matter most** — Lab evals are necessary but not sufficient

### Practical Patterns
- ✅ **Evals detect regressions** — Run regression tests before deploying
- ✅ **Confidence scoring prevents surprises** — Flag uncertain answers
- ✅ **Fallback strategies** — When confidence is low, have a backup plan
- ✅ **Continuous monitoring** — Production health matters more than old test scores

### When You're Stuck
- 📊 Check golden dataset quality first (most common issue)
- 🎯 Measure separately: retrieval vs generation, easy vs hard cases
- 🔄 Compare old vs new on production data (not just lab data)
- 📈 Set up continuous feedback loop (users → failures → improvements)

---

*End of Evals & Reliability Master Guide*

**This guide covers everything from building golden datasets through production monitoring.**  
**Use it as a reference when evaluating, interviewing, or teaching quality assurance for GenAI systems.**
