---
aliases: ["Hybrid Retrieval and RRF"]
tags: [rag, retrieval, fusion, architecture]
sources: [SOTA Agentic RAG Reference Architecture - A System-Design Session Brief.md, Query Modification for a SOTA Agentic RAG System - Taxonomy Benchmarks and a Staged Design.md, The Everything Else Map of SOTA RAG Optimization (2024–2026) - A Decision-Ready Survey for a Greenfield Agentic RAG System.md]
created: 2026-08-27
updated: 2026-08-27
---

# Hybrid Retrieval and RRF

BM25 ∪ dense vector retrieval, fused by **Reciprocal Rank Fusion**. The core of the RAG fast path.

## Why RRF and not weighted scores

RRF operates on **ranks, not raw scores**, so it needs no score-scale calibration between BM25 and cosine similarity.

**This is the architectural mistake most failed production hybrid implementations share**: trying to combine BM25 and cosine with a single weighted-score formula. The scales are incommensurable and the weights never generalize.

Original method: Cormack, Clarke & Büttcher, SIGIR 2009 (DOI 10.1145/1571941.1572114), default **k=60** — *"consistently yields better results than any individual system, and better results than the standard method Condorcet Fuse."*

| Method | Needs calibration | Tuning | When |
|---|---|---|---|
| **RRF** | No (rank-based) | k only (~60) | **Default** — heterogeneous BM25 + dense |
| Weighted score fusion | Yes (fragile) | per-retriever weights | Only when scores are genuinely comparable |
| Relative Score Fusion | Yes (normalized) | — | Weaviate default; when you need score magnitudes |
| Learned reranker | N/A (second stage) | model choice | **Always**, as stage 2 |

Native RRF support: [[OpenSearch]] 2.19 (Feb 2025), [[Qdrant]] 1.10+, Weaviate. Elasticsearch's requires an Enterprise license.

## Expectations, honestly

RRF gives **consistent but modest** gains over the best single retriever — e.g. +0.0094 nDCG@100 on the DAPFAM patent benchmark, though >40% Recall@10 gains over BM25-only on a scientific-code benchmark. The value is robustness across query types, not a headline number.

## Why the hybrid matters for Hungarian specifically

BM25 was found **"competitive"** on Hungarian domain data in the only dedicated Hungarian retrieval study — which independently validates keeping the lexical leg rather than going dense-only. Hungarian is agglutinative, so the BM25 leg benefits materially from lemmatization and analyzer support, while the multilingual dense leg carries the cross-lingual weight. See [[BGE-M3]].

## Feed the legs different queries

**Asymmetric per-leg processing**: extract keywords and entities for the BM25 leg, keep a natural-language query for the dense leg, and **keep the original query for reranking**. The legs reward different query forms. See [[Query Rewriting and Expansion]].

## What this makes redundant

Because the pipeline already does hybrid + RRF + a cross-encoder, **adding multi-query LLM fusion (RAG-Fusion) on top yields diminishing returns** — its recall gains are "largely neutralized after re-ranking and truncation." The hybrid is where the fusion budget is already spent.

## Related

[[Two-Stage Retrieve-Then-Rerank]] · [[Learned Sparse Retrieval]] · [[Qdrant]] · [[OpenSearch]] · [[Query Rewriting and Expansion]]
