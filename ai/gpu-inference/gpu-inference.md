# GPUs for Model Inference

Notes from:
- Docs: [NVIDIA Multi-Instance GPU User Guide](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/) — NVIDIA
- Page: [NVLink & NVLink Switch](https://www.nvidia.com/en-us/data-center/nvlink/) — NVIDIA
- Page: [CUDA GPU compute capability](https://developer.nvidia.com/cuda-gpus) — NVIDIA
- Docs: [NVIDIA device plugin for Kubernetes](https://github.com/NVIDIA/k8s-device-plugin) — NVIDIA (GitHub)
- Docs: [Parallelism and scaling](https://docs.vllm.ai/en/latest/serving/parallelism_scaling.html) — vLLM
- Docs: [Model memory estimator](https://huggingface.co/docs/accelerate/usage_guides/model_size_estimator) — Hugging Face Accelerate
- Docs: [Running Hugging Face models — Setup & prerequisites](https://doc.dataiku.com/dss/latest/generative-ai/huggingface-models/setup.html) — Dataiku (example of a platform's GPU requirements)

Related: [embeddings](../embeddings/embeddings.md) · [RAG](../rag/rag.md)

## Why it matters

Self-hosting models (an [embedding model](../embeddings/embeddings.md#where-the-embedding-model-runs),
a generative LLM) keeps data in-house and works air-gapped — but then **you** have to answer:
does the model fit on the GPU, can several models share a GPU, and what if a model is bigger
than one GPU?

## Prerequisites: compute capability

**Compute capability (CC)** is NVIDIA's version number for a GPU architecture's feature set.
Serving stacks set a floor (e.g. Dataiku's local Hugging Face models require **CC ≥ 7.5**
plus a driver compatible with the required CUDA version).

| CC | Architecture | Example data-center GPUs |
|----|--------------|--------------------------|
| 7.5 | Turing | T4 |
| 8.0 / 8.6 | Ampere | A100, A30 / A10, A40 |
| 8.9 | Ada Lovelace | L40S |
| 9.0 | Hopper | H100, H200, GH200 |

Checklist: **GPU CC ≥ required → driver supports the CUDA version the software needs →
framework (PyTorch/vLLM) built for that CUDA.**

## Does the model fit? Memory sizing

Rule of thumb for **weights**:

```
memory ≈ parameters × bytes per parameter
  float32 = 4 B   float16/bfloat16 = 2 B   int8 = 1 B   int4 = 0.5 B
```

| Model size | fp16 | int8 | int4 |
|------------|------|------|------|
| 0.6B | ~1.2 GB | ~0.6 GB | ~0.3 GB |
| 4B | ~8 GB | ~4 GB | ~2 GB |
| 8B | ~16 GB | ~8 GB | ~4 GB |
| 70B | ~140 GB | ~70 GB | ~35 GB |

Then add overhead: Hugging Face notes inference can need **up to ~20% more** than the
weights alone (activations), and generative LLMs additionally need a **KV cache** that grows
with context length × concurrent requests — often the dominant cost for long contexts.
Serving engines like vLLM pre-allocate a large share of GPU memory for this cache by default,
which is one more reason two models on one GPU collide.

Examples from a platform's docs: a ~4B model fits on a single 24 GB A10; a 70B model wants
2 × 80 GB A100.

## Sharing one GPU between several models

Running two models on the **same, unpartitioned** GPU risks **out-of-memory (OOM)**: each
process grabs memory without knowing about the other (Dataiku's docs flatly warn it "will
likely fail with out-of-memory errors"). Options:

| Technique | What it does | Isolation | Notes |
|-----------|--------------|-----------|-------|
| **One model per GPU** | Simplest | full | Wasteful if the model is small (a 4B embedding model alone on an 80 GB GPU) |
| **Time-slicing** | Processes take turns on the whole GPU | **none** for memory or faults | Kubernetes device plugin can advertise N "replicas" of one GPU |
| **MIG** (Multi-Instance GPU) | Hardware-partitions the GPU into up to **7** isolated instances | **full**: own SMs, L2 cache, memory controllers, memory bandwidth, fault isolation | Ampere and newer data-center GPUs (A100, A30, H100, H200, …) |

### MIG in one picture

```
 One physical GPU (e.g. 80 GB)
 ┌─────────┬─────────┬─────────┬─────────────────────┐
 │ 1g.10gb │ 1g.10gb │ 2g.20gb │      3g.40gb        │
 │embedding│reranker │ small   │   mid-size LLM      │
 │ model   │         │ LLM     │                     │
 └─────────┴─────────┴─────────┴─────────────────────┘
   each slice = its own compute + its own memory → no OOM spill-over, predictable latency
```

Profile names read `<compute slices>g.<memory>gb`. NVIDIA's pitch is "defined quality of
service (QoS) with fault isolation": one tenant can't slow down or crash another. So "one
model per GPU" and MIG don't conflict — **each MIG instance *is* a GPU** from the model's
point of view.

On Kubernetes, the NVIDIA device plugin exposes MIG through a **MIG strategy**:

| Strategy | Exposed as |
|----------|------------|
| `none` | whole GPUs: `nvidia.com/gpu` |
| `single` | all GPUs sliced the same way; slices still requested as `nvidia.com/gpu` |
| `mixed` | per-profile resources, e.g. `nvidia.com/mig-1g.10gb`, `nvidia.com/mig-3g.40gb` |

Watch out: GPU monitoring per MIG instance inside containers is less mature than whole-GPU
monitoring — validate your observability before relying on it.

## When a model is bigger than one GPU: NVLink & parallelism

The opposite problem: a large LLM whose weights + KV cache exceed one GPU.

- **Tensor parallelism** splits every layer across several GPUs, which then exchange data at
  *every* layer — so it needs a very fast GPU-to-GPU link. vLLM's guidance: use it when the
  model is too big for one GPU but fits in one node; set `tensor_parallel_size` = GPUs per node.
- **Pipeline parallelism** gives each GPU/node a slice of the layers; less chatty, used across
  nodes (or on single nodes *without* NVLink, where vLLM recommends preferring it).
- **NVLink** is NVIDIA's direct GPU-to-GPU interconnect (e.g. 900 GB/s per GPU on Hopper,
  1.8 TB/s on Blackwell), far faster than PCIe. **NVLink Switch** connects many GPUs
  all-to-all so a rack behaves like "a single high-performance accelerator".

NVLink does not literally merge memory into one pool; it makes splitting a model across GPUs
fast enough that, in practice, their memory **adds up** for one workload.

### MIG vs NVLink — opposite directions

| | MIG | NVLink (+ tensor parallelism) |
|---|---|---|
| Direction | **one GPU → many** small isolated GPUs | **many GPUs → one** big logical accelerator |
| For | many small models / tenants | one model too large for a single GPU |
| Typical workload | embedding models, rerankers, small LLMs, dev/test | large LLM serving, training, fine-tuning |

Mnemonic: **MIG = Many Instances on one GPU; NVLink = N GPUs Linked into one.**

A mixed fleet often does both: NVLink-connected GPUs for a large LLM, MIG-partitioned GPUs for
the embedding model and other small models. Before planning MIG, **confirm the GPU
generation** — MIG doesn't exist before Ampere.

## Quiz

Self-test on this material: **[AI Retrieval Quiz](https://claude.ai/artifact/Svt3fPZsJ4bmAhjRwB67Q5)**
— shared with the other `ai/` notes (embeddings, vector databases, RAG, GPU inference). Draws 10
random questions from a pool of 30 each time, with at least one per topic, so it's reusable.
Supports skip / previous / jump-to-question navigation, and keeps your progress in the
browser if you refresh.

## Takeaways

- Check compute capability and driver/CUDA before anything else.
- Weights ≈ params × bytes/param, plus ~20% and a KV cache for LLMs.
- Never co-host models on an unpartitioned GPU without accounting for memory; MIG gives hard isolation.
- Big model → NVLink + tensor parallelism; many small models → MIG.
