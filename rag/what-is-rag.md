# Retrieval-Augmented Generation (RAG)

## What is RAG?

Retrieval-Augmented Generation (RAG) is a technique that enhances LLM responses by injecting relevant external knowledge at inference time — before the model generates a response.

Instead of relying solely on knowledge baked into model weights during training, RAG retrieves up-to-date or domain-specific context from an external knowledge base and includes it in the prompt. The model then generates a response grounded in that retrieved context.

**The core problem RAG solves:**
- LLMs have a knowledge cutoff and cannot access live or proprietary data
- Fine-tuning is expensive and doesn't scale well for frequently changing data
- RAG provides a cheaper, faster, and more maintainable path to grounding model outputs in specific knowledge

---

## Architecture

A RAG pipeline has two main phases: **indexing** (offline) and **retrieval + generation** (online).

```
┌─────────────────────────────────────────────────────┐
│                  INDEXING (Offline)                 │
│                                                     │
│  Raw Documents                                      │
│       │                                             │
│       ▼                                             │
│  [Chunking]  ──► split into overlapping chunks      │
│       │                                             │
│       ▼                                             │
│  [Embedding Model]  ──► chunk → vector              │
│       │                                             │
│       ▼                                             │
│  [Vector Store]  ──► store (vector, metadata, text) │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│              RETRIEVAL + GENERATION (Online)        │
│                                                     │
│  User Query                                         │
│       │                                             │
│       ▼                                             │
│  [Embedding Model]  ──► query → vector              │
│       │                                             │
│       ▼                                             │
│  [Vector Store]  ──► similarity search → top-k docs │
│       │                                             │
│       ▼                                             │
│  [Prompt Assembly]                                  │
│  "Answer using the context below:\n{docs}\n{query}" │
│       │                                             │
│       ▼                                             │
│  [LLM]  ──► grounded response                       │
└─────────────────────────────────────────────────────┘
```

### Key Components

#### 1. Document Chunking
Documents are split into smaller chunks before embedding. Chunk strategy matters significantly:
- **Fixed-size chunking** — split by character/token count with overlap (simple, common)
- **Sentence/paragraph chunking** — respect natural boundaries (better semantic coherence)
- **Recursive splitting** — try paragraph → sentence → word boundaries in order
- **Semantic chunking** — embed sentences and split where meaning shifts (expensive but high quality)

> Chunk size trade-off: smaller chunks = more precise retrieval; larger chunks = more context per result.

#### 2. Embedding Model
Converts text into dense vector representations that capture semantic meaning. Similar texts have vectors close together in embedding space.

Common choices:
- `text-embedding-3-small` / `text-embedding-3-large` (OpenAI)
- `embed-english-v3.0` (Cohere)
- `all-MiniLM-L6-v2` (open-source, fast, good baseline)
- `nomic-embed-text` (open-source, strong performance)

#### 3. Vector Store
Stores embeddings and supports approximate nearest neighbor (ANN) search.

| Store | Notes |
|---|---|
| **Pinecone** | Managed, scalable, production-ready |
| **Weaviate** | Open-source, hybrid search support |
| **Qdrant** | Open-source, Rust-based, fast |
| **ChromaDB** | Lightweight, great for local/dev |
| **pgvector** | Postgres extension, good if already on Postgres |
| **FAISS** | Meta's in-memory library, no persistence |

#### 4. Retrieval
Query is embedded and compared against stored vectors using cosine similarity or dot product. Top-k most similar chunks are returned.

**Retrieval strategies beyond naive k-NN:**
- **Hybrid search** — combine dense (semantic) + sparse (BM25 keyword) retrieval, re-rank results
- **MMR (Maximal Marginal Relevance)** — reduce redundancy in retrieved chunks
- **Multi-query retrieval** — LLM generates multiple query variants to improve recall
- **HyDE (Hypothetical Document Embeddings)** — embed a hypothetical answer and retrieve by that

#### 5. Prompt Assembly & Generation
Retrieved chunks are injected into the prompt as context. The LLM is instructed to answer using that context.

---

## RAG vs. Fine-Tuning

| | RAG | Fine-Tuning |
|---|---|---|
| Knowledge update | Easy — update the index | Hard — retrain required |
| Cost | Low (inference-time retrieval) | High (training compute) |
| Transparency | Can cite sources | Opaque |
| Hallucination risk | Lower (grounded in retrieved text) | Higher |
| Best for | Dynamic, changing, or proprietary data | Changing model behavior/tone/format |

RAG and fine-tuning are complementary — you can fine-tune a model and use RAG on top of it.

---

## Use Cases

### Enterprise Knowledge Management
Internal wikis, documentation, HR policies, compliance docs — employees can query in natural language instead of searching manually.

### Customer Support
Ground a chatbot in product documentation, FAQs, and support tickets. Reduces hallucinations and ensures answers stay current as docs change.

### Code Assistance
Retrieve relevant internal codebase context, API docs, or architectural decision records before generating code suggestions.

### Legal & Compliance
Query contracts, regulations, or case law. Retrieve the exact clause before letting the model interpret it.

### Healthcare
Retrieve patient history, clinical guidelines, or drug interaction data to augment diagnostic assistance tools.

### Financial Analysis
Retrieve earnings reports, filings, or news before generating summaries or analysis.

### Personalized Education
Retrieve course material and student history to generate adaptive explanations and exercises.

---

## Common Failure Modes

| Problem | Cause | Fix |
|---|---|---|
| Retrieval misses relevant chunks | Poor chunking or embeddings | Tune chunk size, try semantic chunking, better embedding model |
| LLM ignores retrieved context | Context too long / noisy | Reduce chunks, re-rank, use cross-encoder |
| Hallucination despite retrieval | LLM supplements gaps with made-up info | Stricter prompt instructions, citation enforcement |
| Stale data | Index not updated | Build a refresh/ingestion pipeline |
| Slow retrieval | Large index, no caching | ANN index tuning, result caching |

---

## Further Reading

- [Original RAG paper — Lewis et al. 2020](https://arxiv.org/abs/2005.11401)
- LangChain and LlamaIndex — the two dominant frameworks for building RAG pipelines
- RAGAS — framework for evaluating RAG pipelines (faithfulness, answer relevance, context recall)
