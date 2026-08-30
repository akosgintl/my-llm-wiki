---
aliases: ["Query Rewriting and Expansion"]
tags: [rag, retrieval, query-understanding, hungarian]
sources: [Query Modification for a SOTA Agentic RAG System - Taxonomy Benchmarks and a Staged Design.md, SOTA Agentic RAG Reference Architecture - A System-Design Session Brief.md]
created: 2026-08-27
updated: 2026-08-27
---

# Query Rewriting and Expansion

The headline finding: **light rewriting reliably helps; heavy expansion often hurts** when your retriever and corpus are already strong.

So: put a cheap, deterministic query-understanding layer on the fast path, and reserve expensive transformations for the agentic path.

## Stage 0 — always on, pre-router, deterministic first

1. **Normalization** — Unicode/casing, spelling correction, acronym and domain-vocabulary mapping via a maintained dictionary. No LLM.
2. **Language detection** (HU/EN — both handled by [[BGE-M3]]).
3. **Conversational rewrite** — condense chat history into a standalone query. **The single highest-ROI transformation for multi-turn chat.** Prompt: *"Rewrite ONLY the final user turn into a standalone query; if already standalone, return it EXACTLY; do not answer; do not add facts; output only the query."*
4. **Self-Query metadata extraction** — an LLM extracts filter predicates ("sci-fi movies after 2000 rated >8" → genre, year, rating) plus a semantic query. Maps directly to [[Neo4j]] parameterized Cypher and [[Qdrant]] payload filters. Use structured/guided JSON decoding, and return `null` when no filter applies.
5. **Asymmetric per-leg split** — keywords/entities → the BM25 leg, natural language → the dense leg, **original query kept for reranking**. NER ([[GLiNER]]2 / [[huBERT]]) feeds both metadata filters and entity anchors that prevent semantic drift.

Cache by (session, turn). It sits **before** the router so the router classifies a clean query — see [[Adaptive RAG Routing]].

## Rewriting: small models suffice

Rewrite-Retrieve-Read (Ma et al., EMNLP 2023, arXiv 2305.14283) trains a **770M T5-large** rewriter with PPO on reader accuracy: hit rate on AmbigNQ **76.4% → 82.2%**, and the small rewriter *"often matched or outperformed a black-box LLM rewriter (ChatGPT)"* on HotpotQA and AmbigNQ.

But **sub-3B base models can fail the task outright** — a context-understanding study (arXiv 2402.00858) found OPT-125M generates the next sentence, rewrites the wrong turn, or copies verbatim. **3–4B instruct is the practical floor**; Qwen 3 4B on [[vLLM]] is the sweet spot for self-hosted rewriting, Self-Query extraction and decomposition.

Successors: RQ-RAG (2024), MaFeRw (AAAI 2025), RAFE (2405.14431), annotation-free RL with verifiable search reward (2507.23242). IBM's Granite 3.2 8B ships a LoRA "query rewrite" intrinsic for decontextualization (arXiv 2504.11704).

## Expansion: skip it globally

**Query2Doc** (EMNLP 2023) boosts BM25 by 3–15% on MS-MARCO/TREC DL but only **~0.4–4% on already-strong dense retrievers**. **HyDE** was designed for the *no-relevance-labels* regime, and its own authors write: *"HyDE with fine-tuned encoder is not the intended usage… less powerful instruction LMs can negatively impact the overall performance of the fine-tuned retriever."*

Evidence that it actively hurts:

- arXiv 2505.12694 — expansion degrades when the LLM lacks knowledge of the query (it invents non-existent entities) and on ambiguous queries (biased expansions narrow coverage).
- Long-tail entity retrieval (Frontiers 2026) — *"Despite its low per-query cost (247 tokens), HyDE degrades performance, resulting in negative return on investment."*
- "Out of Style" (arXiv 2504.08231) — HyDE+rerank improves robustness but *"still lags behind original queries."*
- Enterprise practitioner consensus: synonyms that help consumer search are **"lethal"** in enterprise RAG ("term sheet" ≠ "contract"; "incident" ≠ "outage").

**Prefer corpus-grounded expansion when you need any:** **CSQE** (Lei et al., EACL 2024, arXiv 2402.18031) extracts pivotal sentences from initially-retrieved documents (PRF-style grounding) plus LLM expansions — *"strong performance without any training, especially with queries for which LLMs lack knowledge."* Gate it behind the [[Corrective RAG]] grader.

## RAG-Fusion: also skip by default

RAG-Fusion (arXiv 2402.03367) generates query variants and fuses with RRF; a "Hybrid+Diverse" config reports +19% nDCG@10. But "Scaling RAG with RAG Fusion" (arXiv 2603.02153) found fusion increases raw recall while *"these gains are largely neutralized after re-ranking and truncation"*, benefits are *"largely confined to recall-scarce query regimes"*, and for most enterprise workloads it *"introduces additional system cost without delivering material improvements."*

Since the pipeline already does [[Hybrid Retrieval and RRF]] plus a cross-encoder, this is redundant. Enable it only if Recall@20 is consistently low.

## Hungarian: use multilingual embeddings, not translation

Cross-lingual work (arXiv 2511.19324) finds dense CLIR models *"derive little benefit from document translation"* and that translation *"can even degrade slightly because of translation-induced noise"* — recommending semantic multilingual embeddings over translation-based pipelines, especially for under-resourced languages. (Caveat: it tests *document-side* translation and excludes Hungarian; Finnish is the closest agglutinative proxy at ~67–68 nDCG@10 on MIRACL.)

And do not use a Hungarian-only embedding model: [[huBERT]] was explicitly among the "Weakest models" in the only dedicated Hungarian study. See [[BGE-M3]].

## Failure modes to watch

Query drift (expansion diverges from intent) · over-expansion (excessively long queries) · **entity drift** ("PostgreSQL 17 API" retrieves the wrong version) · multi-turn drift (over-eager conversational rewrites).

**Rewrite plausibility is a poor proxy for retrieval lift.** A/B each transformation holding the rest of the pipeline fixed, measure Recall@k and nDCG@10 separately from faithfulness, and feature-flag each so you can disable regressions.

## Related

[[Adaptive RAG Routing]] · [[Query Decomposition and Multi-Hop]] · [[Corrective RAG]] · [[Hybrid Retrieval and RRF]] · [[BGE-M3]]
