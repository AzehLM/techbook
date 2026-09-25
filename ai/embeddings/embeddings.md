# Embeddings

Notes from:
- Paper: [MTEB: Massive Text Embedding Benchmark](https://arxiv.org/abs/2210.07316) — Muennighoff, Tazi, Magne, Reimers (2022)
- Paper: [Matryoshka Representation Learning](https://arxiv.org/abs/2205.13147) — Kusupati et al. (NeurIPS 2022), and the [Hugging Face blog post on Matryoshka embeddings](https://huggingface.co/blog/matryoshka)
- Docs: [Semantic Textual Similarity](https://www.sbert.net/docs/sentence_transformer/usage/semantic_textual_similarity.html) — Sentence Transformers
- Model card: [Qwen3-Embedding-4B](https://huggingface.co/Qwen/Qwen3-Embedding-4B) — Hugging Face (used as a worked example)
- Docs: [Pooling models](https://docs.vllm.ai/en/latest/models/pooling_models.html) — vLLM
- Docs: [Running Hugging Face models — Setup & prerequisites](https://doc.dataiku.com/dss/latest/generative-ai/huggingface-models/setup.html) and [Embedding and searching documents](https://doc.dataiku.com/dss/latest/generative-ai/knowledge/documents.html) — Dataiku (example of a platform that wraps all of this)

Related: [vector databases](../vector-databases/vector-databases.md) · [RAG](../rag/rag.md) · [GPU inference](../gpu-inference/gpu-inference.md)

## What it is

An **embedding** is a fixed-length list of numbers (a vector) that an **embedding model**
produces from a piece of content (a sentence, a paragraph, an image). The model is trained
so that content with **similar meaning lands close together** in that vector space, even if
the words differ:

```
"How do I reset my password?"      → [0.12, -0.40, 0.07, ...]   ┐ close
"I forgot my login credentials"    → [0.10, -0.38, 0.11, ...]   ┘
"Quarterly revenue grew 4%"        → [-0.55, 0.21, 0.90, ...]     far
```

That turns "find text that means the same thing" into a geometry problem: embed the
question, then look for the nearest vectors. That lookup is the job of a
[vector database](../vector-databases/vector-databases.md), and it's the retrieval half of
[RAG](../rag/rag.md).

| Term | Meaning |
|------|---------|
| **Dense embedding** | Every dimension holds a value; captures meaning ("semantic search"). What people usually mean by "embedding". |
| **Sparse embedding** | Mostly zeros, one dimension per vocabulary token (BM25-like); captures exact keywords. |
| **Dimension** | Length of the vector (e.g. 384, 1024, 2560, 4096). Fixed per model (or per chosen size, see MRL). |
| **Pooling** | How per-token outputs are squashed into one vector (mean, CLS token, last token). |
| **Normalization** | Scaling each vector to length 1, so dot product = cosine similarity. |

## Embedding models vs generative LLMs

They are **two different models** with two different jobs:

| | Embedding model | Generative LLM |
|---|---|---|
| Input → output | text → one vector | text → more text |
| Used for | indexing + searching | writing the answer |
| Typical size | ~0.1B – 8B parameters | ~1B – hundreds of B |
| Called | once per chunk at ingestion, once per query | once per answer |

In a RAG pipeline **they are independent**: you can pair any embedding model with any LLM,
because the LLM only ever sees the retrieved *text*, never the vectors. The real coupling
is elsewhere:

- **The index is tied to the embedding model.** Query vectors and stored vectors must come
  from the **same model** (and the same dimension/normalization), otherwise distances are
  meaningless.
- **Changing embedding model = re-embedding the whole corpus** and rebuilding the index.
  Pick deliberately; it's the stickiest choice in the pipeline.

## Choosing an embedding model

Checklist of criteria, with an example model card filled in:

| Criterion | Why it matters | Example: Qwen3-Embedding-4B |
|-----------|----------------|------------------------------|
| **Retrieval quality** on *your* kind of data | Leaderboards are a start, not an answer — MTEB found that "no particular text embedding method dominates across all tasks" | Top-ranked family on the MTEB multilingual leaderboard at release |
| **Languages** | A mostly-English model degrades on French/other corpora and cross-lingual queries | 100+ languages, incl. code |
| **Context window (max input)** | Anything longer than this is truncated → sets the upper bound on chunk size | 32k tokens |
| **Dimension** | Bigger = more expressive but more storage, RAM and slower search | 32 – 2560 (configurable) |
| **Size / hardware footprint** | Has to fit next to everything else on your GPU(s) | 4B parameters |
| **License** | Commercial use, redistribution | Apache 2.0 |
| **Serving support** | Your stack (vLLM, Transformers, a platform's connector) must actually run it | Supported by vLLM / Transformers |
| **Instruction-aware** | Some models take a task prefix ("Represent this query for retrieval: ...") that boosts quality — and must be used consistently | Yes |

Siblings of the same family often trade quality for footprint (here: 0.6B → 1024 dims,
4B → 2560, 8B → 4096). Starting with a smaller sibling and moving up only if retrieval
quality is insufficient is a reasonable default.

### MTEB in one line

The **Massive Text Embedding Benchmark** scores models across 8 task types (retrieval,
reranking, clustering, classification, STS, …) on 58 datasets in 112 languages (original
paper; the leaderboard has grown since). For RAG, look at the **Retrieval** column in your
language(s), not the overall average.

### Matryoshka embeddings (MRL)

Models trained with **Matryoshka Representation Learning** pack the most important
information into the *first* dimensions, so you can **truncate** a vector (e.g. 2560 → 512)
and keep most of the quality. The HF write-up reports ~98% of performance kept at ~8% of the
size. Useful knob to cut storage and search cost — but again, query and stored vectors must
be truncated the same way.

Mnemonic: like Russian dolls, each smaller vector is nested inside the bigger one.

## Similarity functions (quick view)

Most embedding models are trained for **cosine similarity**. If the model already
**L2-normalizes** its output (many do, and vLLM's `embed` task normalizes by default),
**dot product = cosine** and is cheaper to compute. Details and the other metrics live in the
[vector databases note](../vector-databases/vector-databases.md#distance-metrics).

## Where the embedding model runs

| Option | How | Pros | Cons |
|--------|-----|------|------|
| **Hosted API** | HTTP call to a provider's embedding endpoint | Zero infra, instant start | Data leaves your network; per-token cost; provider can deprecate the model |
| **Self-hosted server** | Serve the model yourself, e.g. vLLM exposes an OpenAI-compatible `/v1/embeddings` endpoint | Data stays in-house; fixed cost | You run GPUs, drivers, upgrades |
| **In-process library** | Load it in the app with Sentence Transformers / Transformers | Simplest for batch jobs | Competes with the app for memory; hard to share |

**Air-gapped** (no outbound Internet) environments rule out the first option entirely: only
self-hosted models work. Platforms reflect this — e.g. in Dataiku, only the local
Hugging Face connection or a custom-built connector plugin can embed without an external API
call; every cloud connector (OpenAI, Azure OpenAI, Bedrock, …) needs outbound access.

Hardware side (GPU prerequisites, fitting several models on one GPU) is covered in
[GPU inference](../gpu-inference/gpu-inference.md).

## Multimodal embeddings

Documents are not just text (scanned pages, diagrams, slides). Two families of approach:

- **Turn images into text, then embed the text** — OCR for text in images, or a
  vision-language model (VLM) that writes a description/summary of each image or page.
  Dataiku's "Embed Documents" recipe works this way: its VLM mode renders pages as images,
  asks the VLM for a summary per range of pages, embeds the summary, and at query time passes
  the matching page images back to the VLM.
- **Multimodal embedding model** — a model that maps images and text into the *same* vector
  space, so an image can be retrieved directly by a text query.

## Quiz

Self-test on this material: **[AI Retrieval Quiz](https://claude.ai/artifact/Svt3fPZsJ4bmAhjRwB67Q5)**
— shared with the other `ai/` notes (embeddings, vector databases, RAG, GPU inference). Draws 10
random questions from a pool of 30 each time, with at least one per topic, so it's reusable.
Supports skip / previous / jump-to-question navigation, and keeps your progress in the
browser if you refresh.

## Takeaways

- Embedding = meaning → vector; nearest vectors = most similar meaning.
- Embedding model and LLM are chosen **independently**; the index is married to the embedding model.
- Choose on retrieval quality *in your language*, context window, dimension, footprint, license, serving support.
- Re-embedding is the cost of changing your mind — budget for it.
