---
aliases: ["Embedding Quantization and MRL"]
tags: [rag, embeddings, storage, cost, optimization]
sources: [The Everything Else Map of SOTA RAG Optimization (2024–2026) - A Decision-Ready Survey for a Greenfield Agentic RAG System.md, Deep-Dive - 10 RAG Optimization Techniques for a Self-Hosted Agentic Stack.md]
created: 2026-08-27
updated: 2026-08-27
---

# Embedding Quantization and MRL

Two orthogonal compressions that stack to **~80% storage cost reduction at near-parity recall** — the cheapest large win in the RAG stack.

## MRL — Matryoshka Representation Learning

arXiv 2205.13147. Nests coarse-to-fine information in the *prefixes* of one embedding, so truncating to the first *d* dimensions yields a usable lower-dimensional vector *"at least as accurate as independently trained low-dimensional representations"* — **with no extra inference cost**.

Evidence: Uber Eats truncated gte-Qwen2 from 1536D to 256D at a cost of **0.002 pp in R@200**, cutting storage to ~16% and index cost to ~23%.

[[Qwen3-Embedding]] supports MRL natively, down to **32 dims**. Set your [[Qdrant]] dimension to the smallest MRL cut that passes your eval.

## Quantization

| Method | Compression | Behavior |
|---|---|---|
| **int8 scalar** | 4× | Qdrant's per-dimension int8 *"preserved cosine to three decimal places"*; uses the 99% value range via `quantile` |
| **binary (1-bit)** | 32× | Works best at ≥1024 dims. **"Significant accuracy degradation" alone** — requires float rescoring |

## Rescoring is the mechanism that makes it safe

The canonical pattern (traceable to the Binary Passage Retriever): **retrieve candidates with compressed codes, then rescore the top-k with full-precision vectors.** Qdrant keeps the original float32 vectors on disk precisely for this.

```json
// collection
{
  "vectors": {"size": 1024, "distance": "Cosine"},
  "quantization_config": {"scalar": {"type": "int8", "quantile": 0.99, "always_ram": true}},
  "hnsw_config": {"on_disk": true}
}
// query
{"params": {"quantization": {"rescore": true, "oversampling": 2.0}}}
```

**Rescoring is ON by default for binary/TurboQuant but NOT for scalar — enable it explicitly.** Reported <2% recall loss with oversampling 3.0 for binary; Cohere embed-english-v2.0 (4096d) hit 0.98 recall@50 with 2× oversampling. Qdrant 1.15+ adds asymmetric query encoding (`query_encoding='scalar8bits'`): 1-bit storage with an 8-bit query.

`always_ram: true` keeps quantized vectors in RAM while the HNSW graph and originals live on disk.

## Combining them

Truncate first (Qwen3-4B 2560→1024 via MRL), **then** int8-quantize the truncated vector, then rescore against the stored higher-precision truncated originals. Storage math: 1024-dim float32 = 4 KB/vector → int8 = 1 KB → binary = 128 bytes, with MRL contributing a further ~2.5×.

## Adoption order and the Hungarian caveat

1. Enable **scalar int8 + rescore + oversampling 2.0** with `always_ram`.
2. Measure recall@k against the float baseline.
3. Only then experiment with MRL truncation and binary quantization.

**Over-aggressive MRL truncation degrades multilingual and Hungarian recall more than English — validate per language.** Decision rule: if binary quantization drops recall by more than 3–5 pp even with rescoring, stay on int8.

Rescoring from on-disk originals also slows queries under I/O pressure, and binary quantization on low-dimensional vectors simply loses too much.

## Related

[[Qdrant]] · [[Qwen3-Embedding]] · [[Two-Stage Retrieve-Then-Rerank]] · [[ColPali]]
