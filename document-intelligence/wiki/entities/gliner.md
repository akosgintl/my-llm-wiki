---
aliases: ["GLiNER"]
tags: [ner, nlp, rag, model]
sources: [SOTA Agentic RAG Reference Architecture - A System-Design Session Brief.md, Query Modification for a SOTA Agentic RAG System - Taxonomy Benchmarks and a Staged Design.md, The Everything Else Map of SOTA RAG Optimization (2024–2026) - A Decision-Ready Survey for a Greenfield Agentic RAG System.md]
created: 2026-08-27
updated: 2026-08-27
---

# GLiNER

Apache-2.0 encoder-based **zero-shot NER** with natural-language labels — you name the entity types you want at inference time rather than training a fixed tagger. Runs on CPU, "orders of magnitude faster than autoregressive LLM-based extraction."

**GLiNER2** unifies NER, classification and relation extraction with schema-driven JSON output, which makes it well-suited to knowledge-graph construction.

## The speed/accuracy trade

On a 30-task query-parsing benchmark GLiNER got **53% fully correct versus GPT-4.1-mini's 100%, but 15× faster (0.08 s vs 1.21 s)**. That is the whole argument: it is the volume path, not the accuracy path.

## The Hungarian caveat

**GLiNER multilingual variants do not list Hungarian and have no published Hungarian benchmark.** Treat it as zero-shot and unvalidated there, and prefer [[huBERT]] or [[HuSpaCy]] for the Hungarian half of a bilingual corpus.

## Where it fits

Make NER an interface with two implementations — a fast GLiNER2/spaCy path for high-volume ingestion and an LLM-extraction path for high-value documents — plus a resolver stage. Its output feeds both the [[Neo4j]] entity graph and the Self-Query metadata filters described in [[Query Rewriting and Expansion]].

## Related

[[huBERT]] · [[HuSpaCy]] · [[GraphRAG and Document Graphs]]
