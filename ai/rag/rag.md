# RAG — Retrieval-Augmented Generation

Notes from:
- Paper: [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401) — Lewis et al. (2020), the paper that named RAG
- Paper: [Lost in the Middle: How Language Models Use Long Contexts](https://arxiv.org/abs/2307.03172) — Liu et al. (2023)
- Article: [Chunking Strategies for LLM Applications](https://www.pinecone.io/learn/chunking-strategies/) — Roie Schwaber-Cohen & Arjun Patel (Pinecone)
- Article: [Introducing Contextual Retrieval](https://www.anthropic.com/news/contextual-retrieval) — Anthropic
- Docs: [Reciprocal rank fusion](https://www.elastic.co/docs/reference/elasticsearch/rest-apis/reciprocal-rank-fusion) — Elastic
- Docs: [Retrieval-Augmented Generation](https://doc.dataiku.com/dss/latest/generative-ai/rag.html), [Concept: Embed recipes and RAG](https://knowledge.dataiku.com/latest/ml-analytics/gen-ai/concept-rag.html), [Knowledge Bank Search tool](https://doc.dataiku.com/dss/latest/agents/tools/knowledge-bank-search.html) — Dataiku
- Article: [5 new Dataiku features to streamline your RAG pipelines](https://www.dataiku.com/stories/blog/features-to-streamline-your-rag-pipelines) — Dataiku blog (Dec 2024)
- Tutorial: [Programmatic RAG with LLM Mesh and LangChain](https://developer.dataiku.com/latest/tutorials/genai/nlp/llm-mesh-rag/index.html) — Dataiku Developer Guide

Related: [embeddings](../embeddings/embeddings.md) · [vector databases](../vector-databases/vector-databases.md) · [GPU inference](../gpu-inference/gpu-inference.md)

## What it is

An LLM only knows what was in its training data, up to its cutoff date, and nothing about
your internal documents. **RAG** fixes that without retraining: at question time you
**retrieve** the most relevant passages from your own corpus and **paste them into the
prompt**, so the model answers *from those passages*.

Lewis et al. framed it as combining **parametric memory** (what's baked into the model's
weights) with **non-parametric memory** (an external, searchable index you can update any
time).

Why bother:

| Problem with a bare LLM | What RAG brings |
|-------------------------|-----------------|
| Doesn't know private/recent data | Answers from your current documents |
| Hallucinates confidently | Grounded in retrieved text; can **cite sources** |
| Retraining is slow and expensive | Update knowledge = re-index documents |
| Access control is all-or-nothing | Retrieval can filter by what the user may see |

## The pipeline: two phases

```
INGESTION (offline, when documents change)
  source docs → extract text → chunk → embed → store in vector DB (+ metadata)

QUERY (online, per question)
  question → embed → retrieve top-k chunks → (rerank) → build prompt → LLM → answer + sources
```

Mnemonic: **"Chop, Embed, Store — then Fetch, Stuff, Generate."**

| Step | Phase | Key decisions |
|------|-------|---------------|
| Extract | ingestion | PDF/Word/HTML parsing, OCR, tables, images ([multimodal](../embeddings/embeddings.md#multimodal-embeddings)) |
| **Chunk** | ingestion | strategy, size, overlap — see below |
| Embed | both | which [embedding model](../embeddings/embeddings.md#choosing-an-embedding-model); same model for docs and queries |
| Store | ingestion | which [vector store](../vector-databases/vector-databases.md#deployment-models), which metadata |
| Retrieve | query | top-k, similarity threshold, filters, hybrid |
| Augment | query | prompt template, ordering of passages, citation format |
| Generate | query | which LLM; it's independent of the embedding model |

## Chunking

Documents are split into **chunks** before embedding because (1) the embedding model has a
max input length, (2) one vector for a whole 50-page document averages away every specific
detail, and (3) you only want to paste the *relevant* part into the prompt.

### The core tension

Chunks must be "big enough to contain meaningful information, while small enough to enable
performant applications" (Pinecone).

| Chunks too **small** | Chunks too **large** |
|----------------------|----------------------|
| Precise match, but the chunk lacks context ("it increased by 4%" — *what* did?) | Keeps context, but the vector blends several topics → fuzzier matches |
| More chunks to store and search | Fewer, heavier chunks eat the prompt's context budget |
| Answer may be split across chunks | Irrelevant text dilutes the answer |

### Strategies (simplest → smartest)

| Strategy | How it splits | Good for | Watch out |
|----------|---------------|----------|-----------|
| **Fixed-size** | Every N tokens/characters, with overlap | Baseline — Pinecone suggests starting here and iterating only if insufficient | Cuts mid-sentence / mid-table |
| **Recursive character** | Tries separators in order — paragraphs `\n\n`, then lines, sentences, words — until chunks fit the size | Good default for prose (LangChain's standard splitter) | Still blind to meaning |
| **Sentence / paragraph** | Natural-language boundaries (NLTK, spaCy) | Clean, readable chunks | Uneven sizes |
| **Document-structure** | Follows the format's own structure: Markdown/HTML headings, PDF sections, LaTeX, code functions | Manuals, specs, docs with headings | Needs a good parser; sections may be huge |
| **Semantic** | Embeds consecutive sentences and cuts where similarity drops (topic shift) | Long flowing text without headings | Experimental; costs embedding calls at ingestion |
| **Contextual** | Keeps a normal split, but an LLM **prepends a short context** to each chunk ("This chunk is from the 2025 annual report, section on revenue…") before embedding and BM25 indexing | Chunks that are meaningless out of context | One LLM call per chunk at ingestion (prompt caching helps) |

### The knobs

- **Chunk size** — in tokens; Pinecone suggests testing a range such as 128–1024. Must stay
  under the embedding model's context window.
- **Overlap** — repeating the tail of chunk *n* at the head of chunk *n+1* (often 10–20%) so
  a fact straddling a boundary appears whole in at least one chunk.
- **Separators** — which boundaries the splitter prefers.
- **Minimum size filter** — drop tiny chunks (page numbers, footers) that only add noise.
- **Metadata per chunk** — source, section title, page, date, permissions: used for filters
  and citations.

**Evaluate, don't guess**: build a small set of real questions with known answers, and
compare retrieval hit rate across chunk sizes/strategies. The right size depends on the
documents *and* the kinds of questions.

## Retrieval settings

| Setting | What it does | Trade-off |
|---------|--------------|-----------|
| **top-k** | How many chunks to fetch | Too low → miss the answer; too high → noise and cost. Anthropic found top-20 worked best in their tests |
| **Similarity threshold** | Drop chunks below a score | Avoids pasting unrelated text when nothing matches; lets the app say "I don't know" |
| **Metadata filters** | Restrict to a source, language, date, or what the user is allowed to see | Pushes access control into retrieval |
| **Reranking** | A second, slower model re-scores the top candidates against the question and keeps the best | Better precision; extra latency |

**Order matters inside the prompt.** "Lost in the Middle" showed that LLMs use information
best when it's at the **beginning or end** of a long context and "significantly degrade"
when it's in the middle — another reason not to stuff 100 chunks in, and to put the best
chunks first (or last).

## RAG variants

### Parent-child (small-to-big) retrieval

Resolves the chunk-size tension by using **two sizes**: search on **small child chunks**
(precise vectors), but hand the LLM the **larger parent** they belong to (full section or
page) as context. Example from Dataiku's blog: legal filings, where the matched clause is
useful only with the surrounding section. Related idea: **sentence-window** retrieval
(match one sentence, return its neighbours).

### Hybrid search

Embeddings are bad at **exact tokens**: error codes, product references, names, acronyms.
Hybrid runs **BM25 keyword search + vector search** and fuses the ranked lists, typically
with **Reciprocal Rank Fusion** (`Σ 1/(k + rank)`, k = 60 in Elasticsearch), which needs no
weight tuning because it only uses ranks. Detail in
[vector databases](../vector-databases/vector-databases.md#filtering-and-hybrid-search).

### Contextual retrieval (Anthropic's numbers)

Reduction in top-20 retrieval failure rate on their benchmarks:

| Technique | Failure-rate reduction |
|-----------|------------------------|
| Contextual embeddings | 35% |
| + contextual BM25 (hybrid) | 49% |
| + reranking | 67% |

Takeaway: the three techniques stack.

### Agentic RAG

Classic RAG does **one** retrieval per question, always. In **agentic** RAG the knowledge base
is a **tool** the agent *decides* to call — maybe zero times, maybe several times with
reformulated queries, possibly alongside other tools (SQL, APIs). Dataiku exposes this as a
"Knowledge Bank Search" tool for its agents. More flexible, less predictable, more LLM
calls.

## Keeping the index fresh

Re-embedding everything on every change is wasteful. Besides the full rebuild baseline,
Dataiku's blog names three incremental modes (the concepts are generic):

| Mode | Behaviour |
|------|-----------|
| **Full rebuild** | Drop and rebuild the whole index (baseline) |
| **Append** | Add new documents only |
| **Smart sync** | Re-embed only changed documents, remove deleted ones |
| **Upsert** | Update existing entries by id, insert new ones |

Remember: changing the **embedding model** always forces a full rebuild.

## Guardrails and evaluation

- **Faithfulness** — is the answer supported by the retrieved passages (no invention)?
- **Answer relevancy** — does it actually address the question?
- **Retrieval quality** — did the right chunk make it into the top-k (hit rate, recall@k)?

Platforms can enforce thresholds on these (e.g. an LLM-as-judge scoring faithfulness and
blocking low-scoring answers), at the cost of extra LLM calls per answer.

## Build options

| Approach | Example | Trade-off |
|----------|---------|-----------|
| Low-code platform | Dataiku: embed recipe → Knowledge Bank → retrieval-augmented LLM → chat app/agent | Fast, governed; limited to what the platform exposes |
| Framework in code | LangChain / LlamaIndex chains (Dataiku's own objects can be wrapped as LangChain objects) | Full control over every step; you maintain the code |
| From scratch | Embedding API + vector DB client + prompt template | Most control, most work |

## Quiz

Self-test on this material: **[AI Retrieval Quiz](https://claude.ai/artifact/Svt3fPZsJ4bmAhjRwB67Q5)**
— shared with the other `ai/` notes (embeddings, vector databases, RAG, GPU inference). Draws 10
random questions from a pool of 30 each time, with at least one per topic, so it's reusable.
Supports skip / previous / jump-to-question navigation, and keeps your progress in the
browser if you refresh.

## Takeaways

- RAG = retrieve relevant chunks, paste them into the prompt, generate a grounded answer.
- Chunking is the most underrated knob: start fixed/recursive, measure, then go structural/semantic/contextual.
- Small chunks to *find*, bigger context to *answer* (parent-child).
- Hybrid + reranking fix most retrieval misses; put the best passages at the edges of the prompt.
- Embedding model and LLM are independent choices; the index is tied to the embedding model.
