---
aliases: ["The Everything Else Map of SOTA RAG Optimization"]
tags: [rag, optimization, survey, security, source]
sources: [The Everything Else Map of SOTA RAG Optimization (2024–2026) - A Decision-Ready Survey for a Greenfield Agentic RAG System.md]
created: 2026-08-27
updated: 2026-08-27
---

# The Everything Else Map of SOTA RAG Optimization

**Source:** The Everything Else Map of SOTA RAG Optimization (2024–2026) - A Decision-Ready Survey for a Greenfield Agentic RAG System.md
**Date ingested:** 2026-08-27
**Type:** decision-ready survey
**Position in the corpus:** thirteenth document (2026-08-26 21:10) — a breadth sweep across ten pipeline stages, covering everything the reference architecture and the query-modification brief left out.

## Summary

Ten stages from ingestion to agent memory, each item labeled **Adopt / Extension point / Skip**, with a prioritized top-ten shortlist, an updated end-to-end architecture diagram, trade-off tables per stage, and a three-phase rollout.

## Key Claims

- **Vision retrieval is the biggest architectural opportunity not yet tapped.** Because the pipeline already runs Visual Document Understanding with layout detection, adding [[ColPali]]/ColQwen2.5 late-interaction retrieval over rendered page images is low-friction and bypasses OCR brittleness for tables, charts and scans. The catch is storage: ~1,030 vectors/page, so token pooling plus two-stage rerank is mandatory.
- **The embedding choice is now outdated** — [[Qwen3-Embedding]] (Apache-2.0, 32k context, 100+ languages, MRL, instruction-aware) is materially stronger than [[BGE-M3]] on multilingual retrieval (70.58 vs 54.6 MMTEB retrieval for 8B vs BGE-M3).
- **Ingestion-time enrichment beats most retrieval-time tricks per dollar.** [[Contextual Retrieval]], late chunking, RAPTOR and parent-child chunking all attack the root cause — context-poor chunks — and compound with everything downstream.
- **Generation and serving economics are where latency and cost are won**: KV-cache reuse, context pruning, quantization, semantic caching. **Long-context "just stuff it" is not a substitute for retrieval at scale** — on LongBench v2, 32k retrieval context beat full 128k without RAG.
- **Reliability is non-negotiable for a bilingual, possibly-untrusted corpus.** [[LettuceDetect]] groundedness plus indirect-prompt-injection defenses belong in v1, not v2. AgentPoison achieves **≥80% attack success at <0.1% poison rate**.

## The prioritized top ten

1. Contextual Retrieval at ingestion · 2. Swap to Qwen3-Embedding · 3. CacheBlend/[[LMCache]] · 4. ColPali vision retrieval · 5. MRL + int8 + rescoring in Qdrant · 6. LettuceDetect gate · 7. Injection defenses · 8. miniCOIL/SPLADE++ sparse leg · 9. [[Provence]] pruning + lost-in-the-middle ordering + citations · 10. Parent-child chunking + RAPTOR.

**Explicitly skipped:** proposition-based indexing, full generator fine-tuning, 1-bit binary quantization without rescoring, graph-heavy memory variants.

## Entities Mentioned

[[Qwen3-Embedding]] · [[BGE-M3]] · [[ColPali]] · [[LettuceDetect]] · [[Provence]] · [[LMCache]] · [[Qdrant]] · [[OpenSearch]] · [[Neo4j]] · [[bge-reranker-v2-m3]] · [[Anthropic]] · [[vLLM]] · [[LangGraph]]

## Concepts Covered

[[Contextual Retrieval]] · [[Chunking Strategies]] · [[RAPTOR Hierarchical Summarization]] · [[Embedding Quantization and MRL]] · [[Learned Sparse Retrieval]] · [[KV-Cache Reuse for RAG]] · [[Context Pruning and Lost-in-the-Middle]] · [[Hallucination Detection in RAG]] · [[Indirect Prompt Injection]] · [[RAG Evaluation]] · [[Permissive Licensing Constraints]]

## Decision thresholds it states

If Qwen3-Embedding beats BGE-M3 by **<2 points** on your Hungarian eval, keep BGE-M3. If binary quantization drops recall **>3–5 pp** even with rescoring, stay on int8. **If OCR quality already yields >~95% correct answers on table/chart questions, defer ColPali.** If LLM-reranker latency pushes p95 past your SLA, keep it off the default path.

## Caveats stated by the source

**No public Hungarian embedding leaderboard exists** — all Hungarian guidance rests on MMTEB aggregates plus one small pre-Qwen3 study. Vendor-reported numbers (Anthropic's failure-rate reductions, LMCache's 4.5×, Qdrant's ColPali speedups) come from the technique authors. Search-R1-style RL headlines are measured against weak baselines. Every number here is "a hypothesis to validate with RAGAS/DeepEval on your bilingual corpus."
