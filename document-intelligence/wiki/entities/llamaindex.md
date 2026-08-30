---
aliases: ["LlamaIndex"]
tags: [rag, orchestration, document-parsing, framework]
sources: [SOTA Agentic RAG Reference Architecture - A System-Design Session Brief.md, Local PDF-Markdown and Document Parsing - State of the Art (August 2026).md, Deep-Dive - 10 RAG Optimization Techniques for a Self-Hosted Agentic Stack.md]
created: 2026-08-27
updated: 2026-08-27
---

# LlamaIndex

MIT. Strongest framework for **ingestion, indexing and retrieval**; commonly paired with [[LangGraph]] for orchestration rather than replacing it.

It shows up in this corpus from two directions at once, which is worth noting because they connect:

## As a parsing vendor

LlamaParse is their commercial document parser; **[[liteparse]]** is its open-sourced (Apache-2.0, Rust) core engine. Their own positioning is the clearest statement of the tiered-parsing idea: *"If you're building an agent…that needs to quickly read a PDF and move on, use LiteParse. If you're building a document intelligence product that needs to nail every table, use LlamaParse."* LlamaIndex has also publicly called [[OmniDocBench]] "saturated."

## As a retrieval framework

Two of its retriever patterns are directly adopted in the reference architecture:

- **Sentence-window** — retrieve a sentence, expand to a surrounding window.
- **AutoMergingRetriever** — build a HierarchicalNodeParser tree, retrieve leaf nodes, and if enough leaves under one parent are retrieved, **merge them into the parent** so the LLM gets consolidated context instead of fragments.

Plus a RaptorPack implementation of [[RAPTOR Hierarchical Summarization]]. See [[Chunking Strategies]].

## Related

[[liteparse]] · [[LangGraph]] · [[Chunking Strategies]] · [[Docling]]
