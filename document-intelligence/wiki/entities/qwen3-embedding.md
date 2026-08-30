---
aliases: ["Qwen3-Embedding"]
tags: [rag, embeddings, multilingual, model]
sources: [The Everything Else Map of SOTA RAG Optimization (2024–2026) - A Decision-Ready Survey for a Greenfield Agentic RAG System.md, Deep-Dive - 10 RAG Optimization Techniques for a Self-Hosted Agentic Stack.md]
created: 2026-08-27
updated: 2026-08-27
---

# Qwen3-Embedding

[[Alibaba Qwen Team]]'s embedding series (arXiv 2506.05176), **Apache-2.0** at all sizes. The proposed replacement for [[BGE-M3]] as primary dense encoder.

## Specs

| Size | Native dims | Layers | MMTEB Mean | MMTEB Retrieval |
|---|---|---|---|---|
| 0.6B | 1024 | 28 | 64.33 | 64.6 |
| 4B | 2560 | 36 | 69.45 | 72.3 |
| 8B | 4096 | 36 | **70.58** | — |

All three: **32K context**, **MRL** support down to **32 dims**, instruction-aware. Per Qwen's blog, *"The 8B size embedding model ranks No.1 in the MTEB multilingual leaderboard (as of June 5, 2025, score 70.58)"* — surpassing Gemini-Embedding. Comparison: multilingual-e5-large-instruct 63.22/57.1, BGE-M3 59.56/54.6.

Trained via weakly-supervised contrastive pre-training on LLM-synthesized data → supervised fine-tuning → slerp-style checkpoint merging. Uses **last-token pooling** (`padding_side='left'`), L2-normalized.

## Instruction format — queries only

```python
def get_detailed_instruct(task, query):
    return f'Instruct: {task}\nQuery:{query}'
```

Documents get **no** instruction. Instruction use yields ~1–5% improvement, and the docs recommend writing instructions **in English even for multilingual corpora**, since most training instructions were English.

**Gotcha:** an instruction-prefix mismatch between indexing and query time silently degrades recall. This is the easiest way to quietly break the system.

## Hungarian

Hungarian is explicitly listed in the GitHub README language table (Uralic family: Finnish, Estonian, Hungarian) among 100+ languages. But **no per-language Hungarian score is published**, and **no Hungarian MTEB retrieval split exists** — Hungarian is only inside the MMTEB aggregate and tasks like WikipediaRetrievalMultilingual/BelebeleRetrieval.

The decision rule is therefore explicit: **if it beats [[BGE-M3]] by fewer than 2 points on your own Hungarian eval, keep BGE-M3** (MIT, native sparse + multi-vector). Run that eval before committing.

## Serving

```bash
vllm serve Qwen/Qwen3-Embedding-8B --runner pooling    # newer vLLM
# or --task embed on vLLM ≈0.8.5
```

Requires `vllm>=0.8.5` and `transformers>=4.51.0` (else `KeyError: 'qwen3'`). [[vLLM]] defaults to last-token pooling + normalization, matching the model. HF examples set `max_length=8192` — a demo default, not the 32K limit.

## Companion

**Qwen3-Reranker** (Apache-2.0, 0.6B/4B/8B) is the strongest open-weight reranker and beats [[bge-reranker-v2-m3]] on multilingual reranking — the natural unified upgrade if you migrate the embedder.

## Related

[[BGE-M3]] · [[bge-reranker-v2-m3]] · [[Embedding Quantization and MRL]] · [[vLLM]]
