---
aliases: ["RAPTOR Hierarchical Summarization"]
tags: [rag, ingestion, chunking, retrieval]
sources: [The Everything Else Map of SOTA RAG Optimization (2024–2026) - A Decision-Ready Survey for a Greenfield Agentic RAG System.md, Deep-Dive - 10 RAG Optimization Techniques for a Self-Hosted Agentic Stack.md]
created: 2026-08-27
updated: 2026-08-27
---

# RAPTOR Hierarchical Summarization

Sarthi et al., ICLR 2024 (arXiv 2401.18059, Stanford). Recursively cluster and summarize chunks into a tree, then retrieve from **both** leaf chunks and higher-level summaries.

## The construction

1. Embed chunks (SBERT)
2. Reduce dimensions (**UMAP**, global then local)
3. **Soft-cluster with Gaussian Mixture Models**
4. Summarize each cluster with an LLM
5. Repeat up the tree

## Retrieval: collapsed tree, not traversal

**Flatten all nodes at all levels into one pool and retrieve by cosine similarity.** The paper found collapsed-tree retrieval **outperforms tree-traversal**.

## The number

> *"by coupling RAPTOR retrieval with the use of GPT-4, we can improve the best performance on the QuALITY benchmark by 20% in absolute accuracy"*

SOTA on multi-step reasoning QA at the time. Integrated in RAGFlow, and available as LlamaIndex's RaptorPack; original implementation `parthsarthi03/raptor`.

## When it helps — and when it does not

| Use RAPTOR | Use flat retrieval |
|---|---|
| holistic / summary questions ("what are the themes") | fact lookup |
| multi-hop needing cross-document synthesis | extractive questions |

Run both and route by query type — which is what the [[Adaptive RAG Routing]] classifier is already for. It complements [[GraphRAG and Document Graphs]] rather than replacing it.

## The operational caveat that decides adoption

**RAPTOR trees do not update incrementally.** Adding documents ideally requires re-clustering and re-summarizing. Plus building the tree costs one LLM summarization pass per cluster per level — non-trivial on a large corpus.

Mitigations: build **per-document subtrees**, batch periodic rebuilds, or segment trees per document/collection so updates stay localized.

## Mapping it onto Neo4j

Model the tree as graph nodes — leaf chunks and summary nodes as `:Chunk` / `:Summary` with `PARENT_OF` / `CHILD_OF` edges and cluster membership as edges — storing embeddings in [[Qdrant]] keyed by [[Neo4j]] node IDs.

This unifies **auto-merging** (walk up `PARENT_OF`) and **RAPTOR** (retrieve summary nodes) into graph traversals, and lets you **localize tree rebuilds to affected subgraphs** on document updates — which is the practical answer to the incremental-update problem.

License: paper CC-BY-4.0; the original repo and the LlamaIndex/LangChain implementations are permissive.

## Related

[[Chunking Strategies]] · [[GraphRAG and Document Graphs]] · [[Neo4j]] · [[Adaptive RAG Routing]]
