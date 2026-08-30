---
aliases: ["Chunking Strategies"]
tags: [rag, ingestion, chunking, retrieval]
sources: [SOTA Agentic RAG Reference Architecture - A System-Design Session Brief.md, The Everything Else Map of SOTA RAG Optimization (2024–2026) - A Decision-Ready Survey for a Greenfield Agentic RAG System.md, Deep-Dive - 10 RAG Optimization Techniques for a Self-Hosted Agentic Stack.md]
created: 2026-08-27
updated: 2026-08-27
---

# Chunking Strategies

Chunking configuration influences retrieval quality **as much as the embedding-model choice** — the Vectara NAACL 2025 study (arXiv 2410.13070). Chroma's evaluation found up to a **9% recall gap** between best and worst strategies.

## The default

**Header-based (structure-aware) splitting first, then recursive character splitting within sections.** Start at **~512 tokens with 10–20% overlap** for Markdown/HTML.

**Semantic chunking's extra cost is often NOT justified** — fixed and recursive chunks frequently match or beat it (Vectara). Do not start there.

Preserve document structure as metadata (header hierarchy, source lineage, position). That metadata **doubles as your [[Neo4j]] document-graph structure** (Document→Section→Chunk) — one artifact, two uses.

## Parent-child / small-to-big / auto-merging — adopt

Index **small** chunks for retrieval precision but return the **parent** (or auto-merge contiguous children) for generation context. Low complexity, robust gains, orthogonal to everything else.

- **[[LlamaIndex]] ParentDocumentRetriever** (via LangChain): e.g. 100-token children → 500-token parents.
- **Sentence-window**: retrieve a sentence, expand to a surrounding window.
- **AutoMergingRetriever**: build a hierarchical node tree; at query time, if enough leaves under one parent are retrieved, **merge them into the parent** so the LLM gets consolidated context instead of fragments.

Practitioner reports cite 15–30% accuracy gains on context-dependent queries (directional, not peer-reviewed). If you already have markdown structure, parent-child is nearly free.

## Enrichment options

| Technique | Verdict |
|---|---|
| **[[Contextual Retrieval]]** | **Adopt** — highest ROI; −49% retrieval failures |
| **Late chunking** | Adopt as complement — within-doc context, no LLM calls |
| **[[RAPTOR Hierarchical Summarization]]** | Adopt selectively for long documents |
| Metadata / synthetic-question enrichment (HyQE, doc2query) | Extension point — cheap to add later; overlaps with contextual retrieval, so measure that first |
| **Proposition-based indexing** | **Skip v1** — decomposing text into atomic factoids improves precision but multiplies index size and ingestion LLM cost, and interacts awkwardly with citation-to-source |

## Keep chunk boundaries stable

An operational constraint that is easy to miss: [[KV-Cache Reuse for RAG]] depends on chunk boundaries and ordering staying stable across queries. Re-chunking invalidates the KV cache — and re-triggers the whole [[Contextual Retrieval]] ingestion pass.

## Incremental updates

Content-hash each chunk; upsert changed chunks into [[Qdrant]] and Neo4j in one transaction with the **Neo4j commit as the consistency gate**; support tombstones for deletes. See [[GraphRAG and Document Graphs]].

Chunk from the OCR block tree, not from the emitted markdown — see [[The OCR-to-RAG Seam]].

## Related

[[Contextual Retrieval]] · [[RAPTOR Hierarchical Summarization]] · [[Neo4j]] · [[KV-Cache Reuse for RAG]]
