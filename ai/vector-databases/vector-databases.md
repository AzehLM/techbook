# Vector Databases & Similarity Search

Notes from:
- Paper: [Efficient and robust approximate nearest neighbor search using Hierarchical Navigable Small World graphs](https://arxiv.org/abs/1603.09320) — Malkov & Yashunin (2016)
- Docs: [Faiss indexes](https://github.com/facebookresearch/faiss/wiki/Faiss-indexes) — Faiss wiki (Meta)
- Docs: [pgvector README](https://github.com/pgvector/pgvector) — pgvector
- Docs: [Chroma clients](https://docs.trychroma.com/docs/run-chroma/clients) and [Configure collections](https://docs.trychroma.com/docs/collections/configure) — Chroma
- Docs: [Indexing overview](https://docs.pinecone.io/guides/index-data/indexing-overview) — Pinecone
- Docs: [dense_vector field type](https://www.elastic.co/docs/reference/elasticsearch/mapping-reference/dense-vector) and [Reciprocal rank fusion](https://www.elastic.co/docs/reference/elasticsearch/rest-apis/reciprocal-rank-fusion) — Elastic
- Docs: [Semantic Textual Similarity](https://www.sbert.net/docs/sentence_transformer/usage/semantic_textual_similarity.html) — Sentence Transformers
- Docs: [Working with Vector stores](https://doc.dataiku.com/dss/latest/generative-ai/knowledge/vector-stores.html) — Dataiku (example of a platform offering both embedded and external stores)

Related: [embeddings](../embeddings/embeddings.md) · [RAG](../rag/rag.md)

## What it is

A **vector database** (or **vector store**) stores [embeddings](../embeddings/embeddings.md)
alongside the original content and metadata, and answers one question fast:

> "Given this query vector, which **k** stored vectors are closest to it?"

That's **k-nearest-neighbour (kNN) search**. A regular database indexes values for
*exact* matches and ranges (`WHERE id = 42`, `price < 10`); a vector database indexes
**geometry** for *similarity*.

Each stored record typically looks like:

| Field | Example |
|-------|---------|
| id | `doc-17#chunk-3` |
| vector | `[0.12, -0.40, …]` (e.g. 1024 floats) |
| text | the chunk itself, returned to the caller |
| metadata | `{source: "handbook.pdf", page: 12, lang: "en", updated: 2026-05-01}` — used for filters |

## Distance metrics

How "close" is measured. The metric must match what the embedding model was trained for
(usually cosine), and must be the same at index time and query time.

| Metric | Formula (intuition) | Range / reading | When |
|--------|---------------------|-----------------|------|
| **Cosine similarity** | angle between vectors, ignores length | 1 = same direction, 0 = unrelated, −1 = opposite | Default for most text embedding models |
| **Dot product (inner product)** | Σ aᵢ·bᵢ — angle *and* length | higher = more similar | Same as cosine when vectors are **normalized** (length 1), and cheaper; some models are trained for raw dot product |
| **Euclidean (L2)** | straight-line distance | 0 = identical, lower = closer | Common default in generic libraries (Faiss `IndexFlatL2`, Chroma's default space `l2`) |
| **Manhattan (L1)** | Σ \|aᵢ − bᵢ\| | lower = closer | Rare for text |
| **Hamming / Jaccard** | bits that differ / set overlap | lower = closer | Binary (quantized) vectors |

Key fact: **for unit-length vectors, cosine, dot product and L2 give the same ranking.**
So normalize once at ingestion and use dot product — Sentence Transformers recommends
exactly this when the model ends with a `Normalize` layer. Faiss has no cosine index; the
recipe is "normalize, then use `IndexFlatIP`".

Watch the **naming trap**: some systems return a *similarity* (higher = better), others a
*distance* (lower = better). pgvector's `<#>` returns the **negative** inner product so it
can sort ascending.

| pgvector operator | Metric |
|-------------------|--------|
| `<->` | L2 distance |
| `<#>` | negative inner product |
| `<=>` | cosine distance (= 1 − cosine similarity) |
| `<+>` | L1 distance |
| `<~>` / `<%>` | Hamming / Jaccard (binary vectors) |

## Exact vs approximate search (ANN)

**Exact (brute-force / "flat")** search compares the query with *every* vector: perfect
recall, cost grows linearly with the corpus. Fine for thousands to low millions of vectors.

**Approximate nearest neighbour (ANN)** indexes pre-organize the vectors so a query only
looks at a small fraction of them. You trade a little **recall** (sometimes missing a true
neighbour) for orders-of-magnitude speed.

**Recall@k** = share of the true top-k neighbours that the index actually returned. It's the
metric every ANN knob moves.

### The main index families

| Index | How it works | Build | Memory | Query speed | Notes |
|-------|--------------|-------|--------|-------------|-------|
| **Flat** | Scan everything | none | vectors only | slow, linear | 100% recall; the baseline to measure others against |
| **IVF** (Inverted File, e.g. `IVFFlat`) | k-means splits the space into `nlist` cells; a query scans only the `nprobe` closest cells | needs a **training** pass on representative data | low overhead | medium–fast | Faiss heuristic: `nlist ≈ C·√n`; add data after training or clusters go stale |
| **HNSW** (Hierarchical Navigable Small World) | Multi-layer proximity graph; search greedily walks from sparse top layers down to the dense bottom layer | slower, no training | highest overhead (graph links) | fastest at high recall | Best speed/recall trade-off in most benchmarks; deletions are awkward (Faiss doesn't support removal) |
| **PQ** (Product Quantization, e.g. `IVFPQ`) | Splits each vector into sub-vectors and stores a short code per sub-vector | training | **very low** (compressed) | very fast | Lossy: lower recall; for very large corpora or tight RAM |

Mnemonic for HNSW: **a motorway map** — the top layer has only a few long-distance
links (motorways), each layer down adds more local roads, and you drive to the region
first, then to the street. That layering is what gives roughly logarithmic search time.

Mnemonic for IVF: **a library with sections** — first pick the few closest shelves
(`nprobe`), then read every book on those shelves.

### The knobs

| Index | Build-time knob | Query-time knob | Effect of raising it |
|-------|-----------------|-----------------|----------------------|
| HNSW | `M` / `m` / `max_neighbors` (links per node, pgvector default 16) and `ef_construction` (pgvector 64, Chroma 100) | `ef_search` (pgvector 40, Chroma 100) | better recall, more memory / slower build / slower query |
| IVF | `lists` / `nlist` | `probes` / `nprobe` (pgvector default 1) | more probes → better recall, slower query |

Quantization is increasingly the default to save RAM: e.g. Elasticsearch's `dense_vector`
uses HNSW with int8 or "BBQ" (binary) quantization by default in recent versions,
advertising 75% to 96% memory reduction.

## Filtering and hybrid search

- **Metadata filtering**: restrict the search to `lang = "fr"` or `source = "HR"`. Pinecone,
  pgvector (plain SQL `WHERE`), Chroma, Elasticsearch all support it. Filters interact with
  ANN indexes (filtering *after* the ANN step can leave you with fewer than k results) — check
  how your store handles it.
- **Namespaces / collections**: hard partitions (per tenant, per project). A query only
  scans one.
- **Hybrid search**: run a **keyword search (BM25)** and a **vector search** and merge the
  two ranked lists. Keywords catch exact identifiers, product codes and rare terms that
  embeddings blur; vectors catch paraphrases. The usual merge is **Reciprocal Rank Fusion**:

  ```
  score(doc) = Σ over result lists  1 / (k + rank_in_that_list)      (k = 60 by default in Elasticsearch)
  ```

  RRF only uses *ranks*, so it needs no tuning to reconcile BM25 scores with cosine scores.
  Hybrid is native in search engines that already had BM25 (Elasticsearch/OpenSearch,
  Azure AI Search) — see [RAG](../rag/rag.md#hybrid-search) for why it matters.

## Deployment models

The biggest architectural choice is **where the store runs**, not which brand it is.

| Model | Examples | What it means |
|-------|----------|---------------|
| **Embedded / in-process library** | Chroma (`PersistentClient`), FAISS, Milvus Lite, Qdrant local mode | Runs inside your application process; index on local disk; nothing to deploy |
| **Self-hosted client-server** | Chroma server, Milvus, Qdrant, Weaviate | A separate service you run and scale; many clients over the network |
| **Extension of an existing DB** | **pgvector** (PostgreSQL), vector fields in Elasticsearch/OpenSearch | Vectors live next to your relational/search data; reuse backups, auth, ops skills |
| **Managed cloud service** | Pinecone (serverless), Azure AI Search, Vertex AI Vector Search, Snowflake Cortex Search, Databricks vector search | Provider runs it; you pay per usage; data sits in that provider's cloud |

### Embedded vs external — the trade-off

| | Embedded (in-process) | External service |
|---|---|---|
| Setup | none | provision, secure, monitor a service |
| Resources | **shares CPU, RAM and disk with the host process** — indexing and search add to the host's load | isolated; sized and scaled independently |
| Reuse | only reachable through the host application | any system can query the same vectors |
| Robustness | lives and dies with the host; backups = host's disk | own HA, backups, replication |
| Best for | proofs of concept, small/medium corpora, single app | production, shared knowledge, large corpora, SLAs |

A common path is **staged**: start **embedded** to prove value quickly, then switch to an
external store (pgvector if you already run PostgreSQL, a search engine if you need hybrid,
a managed service if you're all-in on one cloud) once reuse, scale or robustness demand it.
Because vectors are rebuilt from source data anyway, the migration is "re-run ingestion
into the new target", not a data migration.

Example: Dataiku's **Knowledge Bank** object defaults to embedded Chroma ("does not require
any setup, and provides good performance even for quite a large corpus") and can instead
target pgvector, Elasticsearch/OpenSearch, Pinecone, Azure AI Search, Vertex, etc. through a
configured connection — same pipeline, different place the vectors physically live. Note
Chroma itself *does* have a client-server mode; a platform may only use the embedded one.

### Choosing — quick questions

1. Do I already operate PostgreSQL / Elasticsearch? → extension first (pgvector / dense_vector).
2. Do I need exact-keyword matching too? → a store with native hybrid search.
3. Must data stay on-prem / air-gapped? → rules out managed SaaS.
4. Will other apps query the same vectors? → external service, not embedded.
5. How many vectors × dimensions? → RAM estimate: `n × d × 4 bytes` for float32, plus index overhead (HNSW graph links). 1M × 1024 dims ≈ 4 GB before overhead.
6. Dimension limits: e.g. pgvector indexes `vector` up to 2,000 dims (`halfvec` up to 4,000) — check against your embedding model.

## Quiz

Self-test on this material: **[AI Retrieval Quiz](https://claude.ai/artifact/Svt3fPZsJ4bmAhjRwB67Q5)**
— shared with the other `ai/` notes (embeddings, vector databases, RAG, GPU inference). Draws 10
random questions from a pool of 30 each time, with at least one per topic, so it's reusable.
Supports skip / previous / jump-to-question navigation, and keeps your progress in the
browser if you refresh.

## Takeaways

- Vector DB = kNN over embeddings + metadata + filters.
- Cosine for most text models; normalize and cosine = dot = same ranking as L2.
- Flat is exact; HNSW is the usual ANN default; IVF is cheaper to build; PQ saves memory.
- Hybrid (BM25 + vectors, fused with RRF) fixes exact-term misses.
- Embedded to start, external when you need isolation, reuse or scale.
