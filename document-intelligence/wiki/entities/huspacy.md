---
aliases: ["HuSpaCy"]
tags: [hungarian, ner, nlp, tool]
sources: [SOTA Agentic RAG Reference Architecture - A System-Design Session Brief.md]
created: 2026-08-27
updated: 2026-08-27
---

# HuSpaCy

The Hungarian spaCy pipeline. `hu_core_news_lg` reports **NER F1 ≈ 0.867** and is CPU-friendly; transformer variants ([[huBERT]], XLM-RoBERTa-large) score higher at higher cost.

That CPU-friendliness is the point: it is the **high-volume ingestion** arm of a two-implementation NER interface, with an LLM-extraction path reserved for high-value documents.

| Option | License | Speed | Hungarian |
|---|---|---|---|
| **HuSpaCy** (spaCy) | MIT | fast | `hu_core_news_lg` F1 ≈ 0.867 |
| [[huBERT]] | permissive | medium (GPU) | SOTA (~0.82 F1 on NerKor's harder 30-type scheme) |
| [[GLiNER]]2 | Apache-2.0 | fast (CPU) | not listed / unvalidated |
| LLM extraction | model-dependent | slow | strong, but GPU-expensive |

Note the two F1 figures are not directly comparable — they are measured on different schemes (HuSpaCy's standard types versus NerKor's ~30-type scheme).

Whichever path runs, add a **resolver stage** afterwards: `neo4j-graphrag` ships `SpaCySemanticMatchResolver` and `FuzzyMatchResolver` (RapidFuzz).

## Related

[[huBERT]] · [[GLiNER]] · [[GraphRAG and Document Graphs]] · [[Neo4j]]
