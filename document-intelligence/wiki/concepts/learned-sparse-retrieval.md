---
aliases: ["Learned Sparse Retrieval"]
tags: [rag, retrieval, sparse, multilingual]
sources: [The Everything Else Map of SOTA RAG Optimization (2024–2026) - A Decision-Ready Survey for a Greenfield Agentic RAG System.md, Deep-Dive - 10 RAG Optimization Techniques for a Self-Hosted Agentic Stack.md]
created: 2026-08-27
updated: 2026-08-27
---

# Learned Sparse Retrieval

Replace or augment BM25 with **context-aware term weights**, keeping the inverted-index machinery. Slots straight into the existing sparse leg of [[Hybrid Retrieval and RRF]].

## Two philosophies

**SPLADE** (v2 arXiv 2109.10086, v3 arXiv 2403.06789) uses the MLM head to produce a sparse term-weight vector over the vocabulary with **term expansion** — it adds related terms not present in the text — plus learned weighting, controlled by FLOPS regularization for sparsity. SPLADE-v3 is *"statistically significantly more effective than both BM25 and SPLADE++"*, >40 MRR@10 on MS MARCO dev, +2% BEIR out-of-domain.

**miniCOIL** ([[Qdrant]]) is built on the BM25 formula: it **keeps exact term matching** but attaches a per-token 4D contextual vector so the same surface term is weighted by meaning — ranking "apple cutter" above "apple charger" for "apple slicer". It does **not** expand to new terms. Requires the IDF modifier enabled in Qdrant. Model: `Qdrant/minicoil-v1` via FastEmbed. It is Qdrant's current recommendation for new sparse projects.

The trade-off is index size: **expansion inflates postings** (larger index, higher query cost); miniCOIL stays close to BM25 index size because it does not expand.

## The Hungarian constraint

**Most SPLADE and neural-sparse models are English-only.** That is the deciding factor here.

Realistic Hungarian options:

1. **`opensearch-neural-sparse-encoding-multilingual-v1`** (Apache-2.0, **15 languages**, MIRACL-evaluated, arXiv 2411.04403) — OpenSearch's first multilingual neural sparse model. **Verified 2026-08-27: Hungarian is NOT among the 15.** The declared set is `bn te es fr id hi ru ar zh fa ja fi sw ko en` (HF model card) — Finnish is the only agglutinative proxy present, and there is no Central or Eastern European language at all. This **eliminates the OpenSearch learned-sparse path for Hungarian** and leaves [[BGE-M3]] sparse weights as the only realistic multilingual sparse leg.
2. **[[BGE-M3]]'s sparse weights** — broad language coverage, and it produces dense + sparse + multivector in a single pass.

English-only SPLADE-v3 is fine for the English half of a bilingual corpus, not the Hungarian half.

## OpenSearch modes

- **bi-encoder** — encode both query and document.
- **doc-only** — encode at ingest, tokenizer-only at query time: **inference-free and faster**. Models `-doc-v3-distill` and `-doc-v3-gte` (pruned at max-value-ratio 0.1 for a smaller index), plus `opensearch-neural-sparse-tokenizer-v1`. All Apache-2.0.

## Adoption discipline

**Keep BM25 as the baseline and A/B against it on Hungarian data before committing.** Sparse-neural multilingual behavior is uneven, and Qdrant explicitly recommends experimenting on your own data. Adopt only if the learned model clearly beats BM25 on your eval.

In [[Qdrant]] a prefetch can run sparse plus dense and fuse server-side; in [[OpenSearch]] run it alongside BM25 and fuse via RRF, then rerank with [[bge-reranker-v2-m3]].

## Related

[[Hybrid Retrieval and RRF]] · [[OpenSearch]] · [[Qdrant]] · [[BGE-M3]]
