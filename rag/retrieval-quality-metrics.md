# Measuring Retrieval Quality in RAG Systems

## Core Metrics

### 1. Recall@K
**What:** Fraction of relevant documents found in the top-K retrieved results.

```
Recall@K = (relevant docs in top-K) / (total relevant docs)
```

- Focuses on **not missing relevant context**
- High recall = fewer missed answers, but may flood context with noise

### 2. Precision@K
**What:** Fraction of retrieved top-K docs that are actually relevant.

```
Precision@K = (relevant docs in top-K) / K
```

- Focuses on **not polluting context with irrelevant chunks**
- High precision = cleaner context, but may miss some relevant info

### 3. MRR (Mean Reciprocal Rank)
**What:** How highly the *first* relevant result is ranked.

```
MRR = mean(1 / rank_of_first_relevant_doc)
```

- Good for question-answering where **one good chunk is enough**
- Doesn't care about what comes after the first hit

### 4. NDCG (Normalized Discounted Cumulative Gain)
**What:** Measures ranking quality — relevant docs ranked higher contribute more.

- Captures **graded relevance** (very relevant > somewhat relevant > irrelevant)
- More nuanced than binary recall/precision
- Computationally heavier to evaluate

### 5. Context Relevance (LLM-as-judge)
**What:** Ask an LLM to score whether retrieved chunks are relevant to the query.

- No ground truth labels needed
- Subjective, model-dependent, expensive at scale
- Part of frameworks like **RAGAS**, **TruLens**

### 6. Answer Faithfulness / Grounding
**What:** Does the generated answer only use information from retrieved context?

- Catches hallucinations introduced at the *generation* step
- Complements retrieval metrics — good retrieval + unfaithful generation = bad RAG

---

## Tradeoffs

| Metric | Strength | Weakness |
|---|---|---|
| Recall@K | Catches missed answers | Ignores noise in context |
| Precision@K | Penalizes noise | Ignores ranking order |
| MRR | Simple, fast to compute | Ignores docs beyond first hit |
| NDCG | Captures rank + graded relevance | Needs labeled relevance judgments |
| LLM-as-judge | No labels needed | Expensive, inconsistent |
| Faithfulness | Catches generation errors | Doesn't measure retrieval directly |

---

## When to Choose What

### Optimize for Recall when:
- Missing a relevant chunk is catastrophic (medical, legal, compliance)
- Your LLM can filter noise well
- Queries are diverse and hard to predict
- Context window is large enough to absorb extra chunks

### Optimize for Precision when:
- Context window is small/costly
- Your task is narrow and well-defined
- Noise confuses the LLM (small models are especially susceptible)
- Latency matters and you can't afford re-ranking

### Use MRR when:
- You have a single-answer Q&A pattern
- You want a quick proxy metric without labeling full ranked lists
- Benchmarking retriever speed/quality at development time

### Use NDCG when:
- Ranking order matters (e.g., showing results in a UI)
- You have graded relevance labels or can generate them
- You're comparing retrievers systematically in evaluation pipelines

### Use LLM-as-judge when:
- You lack labeled datasets (most real projects)
- You want end-to-end RAG pipeline evaluation (RAGAS suite)
- Running offline evals, not production monitoring

### Use Faithfulness when:
- You need to audit for hallucination
- The retrieved docs are good but answers seem wrong
- Building trust/compliance requirements around the system

---

## Practical Hierarchy

```
Start with:  Recall@5 or Recall@10  ← are you finding the right stuff?
Then add:    Precision@K             ← are you adding too much noise?
Then add:    MRR / NDCG             ← is ranking helping or hurting?
End-to-end:  Faithfulness + Answer Correctness (RAGAS-style)
```

**The most common mistake** is only measuring answer quality end-to-end — when answers are wrong, you can't tell if the retriever failed or the generator hallucinated. Measure retrieval independently.
