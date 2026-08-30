---
aliases: ["OpenSearch"]
tags: [rag, retrieval, bm25, sparse, infrastructure]
sources: [SOTA Agentic RAG Reference Architecture - A System-Design Session Brief.md, The Everything Else Map of SOTA RAG Optimization (2024–2026) - A Decision-Ready Survey for a Greenfield Agentic RAG System.md, Deep-Dive - 10 RAG Optimization Techniques for a Self-Hosted Agentic Stack.md, Query Modification for a SOTA Agentic RAG System - Taxonomy Benchmarks and a Staged Design.md]
created: 2026-08-27
updated: 2026-08-27
---

# OpenSearch

Apache-2.0. The BM25 and learned-sparse leg of the hybrid retriever.

**Why it rather than Elasticsearch:** OpenSearch added **native RRF in 2.19 (Feb 2025)**, while Elasticsearch's native RRF requires an Enterprise license and its license is SSPL/Elastic — not OSI-permissive. For a permissive-only stack this settles it.

## Neural sparse models (all Apache-2.0)

Two modes:

- **bi-encoder** — encode both query and document
- **doc-only** — encode the document at ingest, tokenizer-only at query time, i.e. **inference-free and faster**

Models: `opensearch-neural-sparse-encoding-doc-v3-distill` and `-doc-v3-gte` (pruned at max-value-ratio 0.1 for a smaller index), plus `opensearch-neural-sparse-tokenizer-v1`; bi-encoder `-v2-distill`.

## The multilingual model — the Hungarian-relevant one

`opensearch-neural-sparse-encoding-multilingual-v1` (Apache-2.0, **15 languages**, MIRACL-evaluated, arXiv 2411.04403) is OpenSearch's first multilingual neural sparse model. **Verified 2026-08-27: Hungarian is NOT among the 15** (`bn te es fr id hi ru ar zh fa ja fi sw ko en`, HF model card). Do not plan a Hungarian sparse leg on it — see [[Learned Sparse Retrieval]].

This matters because most SPLADE-family models are English-only. The realistic Hungarian learned-sparse options are this model or [[BGE-M3]]'s sparse weights — not English SPLADE-v3. See [[Learned Sparse Retrieval]].

## In the pipeline

Run BM25 (baseline) and/or a learned-sparse model here, dense in [[Qdrant]], fuse via RRF, then rerank with [[bge-reranker-v2-m3]]. Hungarian is agglutinative, so the BM25 leg benefits materially from lemmatization and analyzer support — and BM25 was found "competitive" on Hungarian domain data, which is itself an argument for the hybrid design.

Feed it the **keyword/entity form** of the query, not the natural-language form — see [[Query Rewriting and Expansion]] on asymmetric per-leg processing. Also index the [[Contextual Retrieval]] blurb here, not only in the dense index; contextual BM25 is half of that technique's gain.

## Related

[[Qdrant]] · [[Hybrid Retrieval and RRF]] · [[Learned Sparse Retrieval]] · [[BGE-M3]]
