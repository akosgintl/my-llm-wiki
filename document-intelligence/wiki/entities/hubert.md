---
aliases: ["huBERT"]
tags: [hungarian, ner, embeddings, model]
sources: [SOTA Agentic RAG Reference Architecture - A System-Design Session Brief.md, Query Modification for a SOTA Agentic RAG System - Taxonomy Benchmarks and a Staged Design.md]
created: 2026-08-27
updated: 2026-08-27
---

# huBERT

SZTAKI-HLT's Hungarian BERT. **SOTA for Hungarian NER** — it set records on the Szeged NER corpus, and a huBERT-based NerKor tagger reaches ~0.82 F1 on the ~30-type NerKor+Cars-OntoNotes++ scheme.

## Use it for NER, not for embeddings

This distinction is load-bearing and easy to get wrong:

| Task | Verdict |
|---|---|
| Hungarian **NER** | ✅ SOTA — the right choice, alongside [[HuSpaCy]] |
| Hungarian **retrieval embeddings** | ❌ Explicitly among the *"Weakest models"* (Antal, *Infocommunications Journal* 2025) on the Clearservice domain corpus, behind [[BGE-M3]] |

So: huBERT feeds the entity-extraction and metadata-filter path, while [[BGE-M3]] or [[Qwen3-Embedding]] carries the dense retrieval leg. Reaching for a Hungarian-only embedding model is a documented mistake.

## Where it sits in the pipeline

Entity extraction feeding both **metadata filters** (Self-Query) and **entity anchors** that prevent semantic drift during query rewriting — see [[Query Rewriting and Expansion]] — plus the entity graph in [[Neo4j]].

Prefer it over [[GLiNER]] for Hungarian: GLiNER's multilingual variants do not list Hungarian and have no published Hungarian benchmark.

Other named Hungarian LLMs (PULI, Racka, SambaLingo-Hungarian) lack published retrieval benchmarks entirely.

## Related

[[HuSpaCy]] · [[BGE-M3]] · [[GLiNER]] · [[Hungarian OCR and the Double Acute Test]]
