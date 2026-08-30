---
aliases: ["KV-Cache Reuse for RAG"]
tags: [rag, serving, performance, optimization]
sources: [The Everything Else Map of SOTA RAG Optimization (2024–2026) - A Decision-Ready Survey for a Greenfield Agentic RAG System.md, Deep-Dive - 10 RAG Optimization Techniques for a Self-Hosted Agentic Stack.md]
created: 2026-08-27
updated: 2026-08-27
---

# KV-Cache Reuse for RAG

**The biggest RAG-specific serving win**, and the exact inverse of the OCR-side advice.

## The problem prefix caching does not solve

[[vLLM]]'s automatic prefix caching only helps when the *prefix* matches. In RAG the **retrieved chunks change per query and appear at varying positions**, and they lack cross-attention to preceding text — so precomputed per-chunk KV caches are not directly reusable, and prefill is recomputed every single query.

## CacheBlend's answer

Pre-compute each chunk's KV cache independently, then at serving time:

1. **reuse all chunks' KV regardless of position**, and
2. **selectively recompute** the KV of a small subset of *cross-attention-deviant* tokens — those whose attention most changes given the other chunks — restoring full-prefill quality.

Reported: *"minimal loss in quality compared with full KV recompute, with 5%–18% selective recompute ratio"*, **TTFT reduced 2.2–3.3× and throughput increased 2.8–5×**. Because the small recompute pipelines with KV retrieval, cheaper storage tiers can hold the cache without adding latency. See [[LMCache]].

## Note the contrast with OCR

| Workload | Prefix caching |
|---|---|
| OCR | **`--no-enable-prefix-caching`** — requests share no prefix; the cache only costs memory |
| RAG generation | **Enable it** — shared system prompts are a free win, and it is the prerequisite for LMCache |

The same flag, opposite answers. Which is why per-workload serving profiles matter more than a house style. See [[vLLM Continuous Batching and Concurrency Sizing]].

Prefix caching is also what makes self-hosted [[Contextual Retrieval]] cheap: put the whole document as a shared prefix and per-chunk generation reuses its KV.

## Adoption order

1. `--enable-prefix-caching` — free.
2. LMCache CPU offload.
3. CacheBlend blending — **only after measuring answer-quality parity**.

## The operational dependency

**Chunk ordering and boundaries affect reuse** (see CacheWeaver). Keep chunk boundaries stable — which means a re-chunking decision now has a serving-cost consequence, not just an ingestion one. See [[Chunking Strategies]].

## Related

[[LMCache]] · [[vLLM]] · [[Chunking Strategies]] · [[Context Pruning and Lost-in-the-Middle]]
