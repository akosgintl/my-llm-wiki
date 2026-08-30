---
aliases: ["Query Modification for Agentic RAG"]
tags: [rag, query-understanding, hungarian, retrieval, source]
sources: [Query Modification for a SOTA Agentic RAG System - Taxonomy Benchmarks and a Staged Design.md]
created: 2026-08-27
updated: 2026-08-27
---

# Query Modification for Agentic RAG

**Source:** Query Modification for a SOTA Agentic RAG System - Taxonomy Benchmarks and a Staged Design.md
**Date ingested:** 2026-08-27
**Type:** taxonomy and staged design
**Position in the corpus:** twelfth document (2026-08-26 20:53) — deepens one slot of the reference architecture: everything that happens to a query before retrieval.

## Summary

A nine-part taxonomy of query modification techniques with benchmark evidence for each, a staged design placing cheap transformations pre-router and expensive ones on the agentic branch only, a trade-off table, prompt patterns, and an explicit "what to skip and why" list.

## Key Claims

- **Light rewriting reliably helps; heavy expansion often hurts.** Put a cheap deterministic query-understanding layer on the fast path and reserve decomposition, step-back and HyDE for the agentic path.
- **Small rewriters suffice.** Rewrite-Retrieve-Read (Ma et al., EMNLP 2023) trains a 770M T5-large rewriter that lifted AmbigNQ hit rate 76.4% → 82.2% and *"often matched or outperformed a black-box LLM rewriter (ChatGPT)."* But sub-3B base models can fail the task outright — **3–4B instruct is the practical floor.**
- **Conversational rewriting is the single highest-ROI transformation for multi-turn chat.** Prompt it to rewrite *only if context-dependent*, otherwise return verbatim.
- **Skip global HyDE/Query2Doc and always-on RAG-Fusion.** Query2Doc gives +3–15% on BM25 but only ~0.4–4% on strong dense retrievers; HyDE's own authors note fine-tuned encoders gain little and can degrade; long-tail entity retrieval measured **negative ROI** for HyDE. RAG-Fusion's recall gains are *"largely neutralized after re-ranking and truncation."*
- **Prefer corpus-grounded expansion (CSQE) over vanilla HyDE**, gated by the CRAG relevance grader.
- **Decomposition is +36.7% MRR@10 on multi-hop — and hurts precise single-hop queries.** Step-back gives +7–27% on reasoning-intensive tasks but is minimal on arithmetic.
- **For Hungarian, rely on [[BGE-M3]] multilingual embeddings, not query translation.** The only dedicated Hungarian study (Antal, *Infocommunications Journal* 2025) found BGE-M3 and XLM-RoBERTa highest at MRR 0.90, with **[[huBERT]] explicitly among the "Weakest models."** Cross-lingual evidence shows dense encoders derive little benefit from translation, which adds noise and latency.
- **Asymmetric per-leg processing:** keywords/entities to the BM25 leg, natural language to the dense leg, **original query kept for reranking**.
- **Rewrite plausibility is a poor proxy for retrieval lift** — A/B each transformation in isolation, with confidence intervals.

## Entities Mentioned

[[BGE-M3]] · [[huBERT]] · [[GLiNER]] · [[Qdrant]] · [[OpenSearch]] · [[Neo4j]] · [[bge-reranker-v2-m3]] · [[LangGraph]] · [[vLLM]] · [[Qwen Model Family]] (Qwen 3 4B as the rewriter)

## Concepts Covered

[[Query Rewriting and Expansion]] — this document is its primary source · [[Query Decomposition and Multi-Hop]] · [[Adaptive RAG Routing]] · [[Corrective RAG]] · [[Hybrid Retrieval and RRF]]

## Failure modes it names

Query drift · over-expansion · **entity drift** ("PostgreSQL 17 API" retrieving the wrong version) · multi-turn drift from over-eager conversational rewrites.

## Caveats stated by the source

The Hungarian retrieval evidence rests substantially on **a single study with a small 50-question domain set** — directional, validate on your own corpus. The cross-lingual paper tests *document-side* translation and excludes Hungarian (Finnish is the closest agglutinative proxy). Hungarian MTEB coverage is classification-only, with no retrieval tasks — the eval harness is non-optional.
