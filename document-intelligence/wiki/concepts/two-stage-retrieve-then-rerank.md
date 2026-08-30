---
aliases: ["Two-Stage Retrieve-Then-Rerank"]
tags: [rag, retrieval, reranking, architecture]
sources: [SOTA Agentic RAG Reference Architecture - A System-Design Session Brief.md, The Everything Else Map of SOTA RAG Optimization (2024–2026) - A Decision-Ready Survey for a Greenfield Agentic RAG System.md, Deep-Dive - 10 RAG Optimization Techniques for a Self-Hosted Agentic Stack.md]
created: 2026-08-27
updated: 2026-08-27
---

# Two-Stage Retrieve-Then-Rerank

**The correct retrieval architecture is two-stage**, and almost everything else in the pipeline is calibrated against it.

1. **First stage — broad recall.** Hybrid BM25 ∪ dense fused by RRF into a candidate pool of **top-100 to top-200**. See [[Hybrid Retrieval and RRF]].
2. **Second stage — precision.** A **cross-encoder reranker** on that shortlist, down to the final ~5–10 chunks.

A SemEval-2026 hybrid RAG system used exactly this: RRF (k=10) → top-200 → [[bge-reranker-v2-m3]] rerank of top-100 → optional α-weighted rescoring. [[Contextual Retrieval]]'s pipeline is the same: retrieve top-150 by RRF → rerank → top-20 into the prompt.

## Why the second stage changes what else is worth doing

The reranker **absorbs** several techniques that look good in isolation:

- **RAG-Fusion / multi-query** — recall gains "largely neutralized after re-ranking and truncation."
- **Query expansion (HyDE, Query2Doc)** — +3–15% on BM25, but only ~0.4–4% on already-strong dense retrievers, and it can *hurt*.
- **Cheap first-stage compression** — aggressive quantization is safe because rescoring and reranking recover the ordering.

The rule: **measure every upstream trick with the reranker in place**, never in isolation. See [[Benchmark Saturation]].

## The same shape appears elsewhere

- **[[ColPali]] vision retrieval**: prefetch on pooled page vectors → exact MaxSim rerank on full multivectors (13× faster, NDCG@20 0.952).
- **[[Embedding Quantization and MRL]]**: retrieve candidates with compressed codes → rescore top-k with full-precision vectors (the Binary Passage Retriever pattern).

Cheap-and-broad first, expensive-and-narrow second, is the general form.

## Reranker as a routed interface

Keep [[bge-reranker-v2-m3]] on the default path. Make the reranker **swappable**, and route only hard or ambiguous queries — identified by the [[Adaptive RAG Routing]] classifier — to an LLM/reasoning reranker (RankZephyr, Rank-R1, Qwen3-Reranker) that costs 10–100× the latency. If that pushes p95 past your SLA, keep it off the default path.

ColBERT-style late interaction is the low-tail-latency option (p50 ~23 ms versus a cross-encoder's p99.9 >21 s at 40 QPS) and shines even more as a *first-stage* retriever.

## Related

[[Hybrid Retrieval and RRF]] · [[bge-reranker-v2-m3]] · [[Adaptive RAG Routing]] · [[Context Pruning and Lost-in-the-Middle]]
