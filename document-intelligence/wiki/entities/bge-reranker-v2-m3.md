---
aliases: ["bge-reranker-v2-m3"]
tags: [rag, reranking, multilingual, model]
sources: [SOTA Agentic RAG Reference Architecture - A System-Design Session Brief.md, Query Modification for a SOTA Agentic RAG System - Taxonomy Benchmarks and a Staged Design.md, The Everything Else Map of SOTA RAG Optimization (2024–2026) - A Decision-Ready Survey for a Greenfield Agentic RAG System.md]
created: 2026-08-27
updated: 2026-08-27
---

# bge-reranker-v2-m3

Apache-2.0 cross-encoder reranker, multilingual (covers Hungarian). The recommended self-host default for stage 2 of [[Two-Stage Retrieve-Then-Rerank]].

## The alternatives, and why they lose

| Model | License | Latency | Verdict |
|---|---|---|---|
| **bge-reranker-v2-m3** | Apache-2.0 | medium | Default |
| ColBERTv2 (late interaction) | Apache-2.0 | very low (p50 ~23 ms vs a cross-encoder's p99.9 >21 s at 40 QPS) | Best when tail latency matters; shines more as a *first-stage retriever* than a pure reranker |
| Qwen3-Reranker | Apache-2.0 | medium | Beats bge-reranker-v2-m3 on multilingual reranking — the natural unified upgrade |
| Jina Reranker v2 multilingual | **CC-BY-NC-4.0** | low | **Avoid** — non-commercial weights |
| Cohere Rerank | managed API | — | Not self-hostable |
| RankZephyr / Rank-R1 / Rank-K / ReasonRank | varies | **10–100× higher** | Better on reasoning-heavy benchmarks (BRIGHT, R2MED); listwise is O(n) with sliding-window/tournament sort |

Note `bge-reranker-v2-gemma` variants carry the Gemma license, not Apache-2.0.

## The architectural rule

**Make the reranker a swappable interface.** Keep bge-reranker-v2-m3 on the default path, and route only hard or ambiguous queries to an LLM/reasoning reranker via the [[Adaptive RAG Routing]] complexity classifier. If LLM-reranker latency pushes p95 past your SLA, keep it off the default path entirely.

## Why it also matters upstream

Reranking is what neutralizes several fashionable techniques. [[Query Rewriting and Expansion]] notes that RAG-Fusion's recall gains are "largely neutralized after re-ranking and truncation" — because a good reranker already recovers what multi-query fusion was adding. Adding expansion on top of a strong reranker mostly buys cost.

It is also the third component of [[Contextual Retrieval]]'s headline number: contextual embeddings alone cut retrieval failure 35%, plus contextual BM25 49%, **plus reranking 67%**.

Feed it the **original** query, not the rewritten or expanded one.

## Related

[[Two-Stage Retrieve-Then-Rerank]] · [[Hybrid Retrieval and RRF]] · [[Qwen3-Embedding]] · [[Contextual Retrieval]]
