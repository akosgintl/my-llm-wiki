---
aliases: ["SOTA Agentic RAG Reference Architecture"]
tags: [rag, agentic, architecture, graph, source]
sources: [SOTA Agentic RAG Reference Architecture - A System-Design Session Brief.md]
created: 2026-08-27
updated: 2026-08-27
---

# SOTA Agentic RAG Reference Architecture

**Source:** SOTA Agentic RAG Reference Architecture - A System-Design Session Brief.md
**Date ingested:** 2026-08-27
**Type:** system-design brief
**Position in the corpus:** eleventh document (2026-08-26 20:39) — **the pivot from OCR to retrieval.** It explicitly reserves a slot for the PDF→Markdown/OCR pipeline the earlier documents designed.

## Summary

A "simple-first" reference architecture: hybrid BM25+dense retrieval fused by RRF, then a cross-encoder reranker, wrapped in an adaptive router that escalates only complex queries into an agentic loop. Neo4j is used as a lightweight document graph, not an ontology. Includes a concrete Neo4j schema, trade-off tables for every component slot, and a four-stage rollout.

## Key Claims

- **Build a modular "simple-first" hybrid core and wrap it in an adaptive router.** Multi-hop agentic retrieval delivers large gains on genuinely compositional questions (MuSiQue ~8.2 → ~39–44 EM) but adds **~2–3× latency** for little gain on single-fact lookups.
- **The reference paper** (Wedge et al., arXiv 2606.05901, Newcastle University) shows a *simple* document graph queried by **pre-built Cypher tools** more than halved refusals and hallucinations on MoNaCo with only a modest token increase — explicitly to avoid the failure and security risk of LLM-generated queries.
- **RRF is the right default fusion method** because it operates on ranks, not scores. Trying to combine BM25 and cosine with a single weighted-score formula is *"the architectural mistake most failed production hybrid implementations share."*
- **Keep [[Qdrant]] for vectors and [[Neo4j]] for relationships, joined by shared IDs** — dedicated vector stores deliver ~10× lower latency than Neo4j's built-in index.
- **Keep ingestion decoupled from retrieval behind clean interfaces** so the future OCR pipeline drops in without touching the query path.
- **For Hungarian, standardize on [[BGE-M3]] or [[multilingual-e5-large-instruct]] embeddings and [[huBERT]]/[[HuSpaCy]] NER.** GLiNER does not list Hungarian.
- **Chunking configuration influences retrieval quality as much as the embedding-model choice** — and semantic chunking's extra cost is often not justified.
- Adopt agentic patterns in ROI order: routing → CRAG → Self-RAG → iterative loops.

## Entities Mentioned

[[Qdrant]] · [[Neo4j]] · [[OpenSearch]] · [[LangGraph]] · [[LlamaIndex]] · [[BGE-M3]] · [[multilingual-e5-large-instruct]] · [[bge-reranker-v2-m3]] · [[huBERT]] · [[HuSpaCy]] · [[GLiNER]]

## Concepts Covered

[[Hybrid Retrieval and RRF]] · [[Two-Stage Retrieve-Then-Rerank]] · [[Adaptive RAG Routing]] · [[Corrective RAG]] · [[GraphRAG and Document Graphs]] · [[Text2Cypher]] · [[Chunking Strategies]] · [[RAG Evaluation]] · [[Permissive Licensing Constraints]]

## Staged plan

**Stage 0 (weeks 1–4)** fast-path MVP: chunking, BGE-M3 on vLLM, Qdrant + OpenSearch, RRF, rerank to top-8, citations; Neo4j document graph in parallel; RAGAS from day one. **Stage 1 (5–8)** adaptive router + CRAG grader. **Stage 2 (9–12)** agentic multi-hop branch with parameterized Cypher tools. **Stage 3** NER→entity graph, web fallback, MS GraphRAG offline reports.

## Caveats stated by the source

Several comparison points come from vendor blogs or practitioner posts, not peer-reviewed benchmarks. **GraphRAG evaluation is contested** — a 2025 study found reported gains partly stem from LLM-judge biases that vanish when corrected. Hungarian embedding quality is weaker than for high-resource languages even for the best models. Licenses to watch: SearXNG AGPL, Neo4j Community GPLv3, Elasticsearch SSPL, Jina reranker v2 CC-BY-NC.
