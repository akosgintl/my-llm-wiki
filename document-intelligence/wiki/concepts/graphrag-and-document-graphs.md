---
aliases: ["GraphRAG and Document Graphs"]
tags: [rag, graph, retrieval, architecture]
sources: [SOTA Agentic RAG Reference Architecture - A System-Design Session Brief.md, The Everything Else Map of SOTA RAG Optimization (2024–2026) - A Decision-Ready Survey for a Greenfield Agentic RAG System.md]
created: 2026-08-27
updated: 2026-08-27
---

# GraphRAG and Document Graphs

**Use a lightweight document/lexical graph, not a heavyweight ontology.** Start simple; add entity structure only when the query log demands it.

## The reference result

Wedge, Stutter, Dixon & Cała (National Innovation Centre for Data, Newcastle University), *"Reducing Hallucinations in Complex QA using Simple Graph-based RAG"*, arXiv 2606.05901 (v2, 22 Jul 2026).

Their design is deliberately *lightweight*: rather than an LLM-constructed rich knowledge graph, a **simple document graph** — document titles, section titles, constituent paragraphs, with links between paragraphs and documents — in [[Neo4j]], with the agent given **pre-written Cypher-backed tools** rather than free-form Cypher generation, explicitly to *"relieve the agent from the task of generating valid data queries – a potential source of failure and security vulnerability."*

Evaluated on **MoNaCo** (1,315 human-written complex questions), vector+graph RAG versus vector-only and zero-shot:

- *"more than halved the proportion of complex questions that the agent refused to answer"*
- increased precision and recall of factual correctness
- *"can halve the number of hallucinated answers"*
- highest fine-grained truthfulness score
- *"all with only a modest increase in token usage"*

(Preprint, CC BY-NC-SA 4.0; the evaluation KG is promised on publication.)

## The framework landscape

| Framework | License | Best for | Index cost |
|---|---|---|---|
| **Simple document graph** (ref paper) | your code | structure-aware retrieval, security | **very low** |
| **HippoRAG 2** | Apache-2.0 | associative/multi-hop **without hurting single-fact** | low |
| LightRAG | MIT | entity-relationship queries, incremental updates | low |
| Microsoft GraphRAG | MIT | global/cross-document summarization | **high** |
| neo4j-graphrag-python | Apache-2.0 | first-party KG build + retrievers | medium |

**HippoRAG 2** (Gutiérrez et al., ICML 2025, "From RAG to Memory", arXiv 2502.14802): dual-node KG (phrase + passage nodes) + Personalized PageRank + LLM triple filtering, *"achieving a 7% improvement in associative memory tasks over the state-of-the-art embedding model while also exhibiting superior factual knowledge and sense-making memory capabilities."* Crucially it improves multi-hop **without degrading single-fact** performance, and is cheaper to index than MS GraphRAG, RAPTOR or LightRAG. Its reference implementation pairs a vector store with Neo4j — the same split recommended here.

## The verdict

1. **Adopt the simple document graph first** — proven, cheap, and secure via curated Cypher tools.
2. **Add HippoRAG-2-style entity/PPR retrieval as the GraphRAG extension point** if multi-hop associative queries become important.
3. **Treat MS GraphRAG as an optional offline "global summary report" generator**, not the core retrieval path.

## Where graphs do NOT help

WildGraphBench (2026) found *"flat baselines like NaiveRAG remain competitive on single-fact retrieval"* and that GraphRAG *"can be more expensive than NaiveRAG or BM25 without clear gains for single-fact lookup."*

And the evidence base itself is contested: **a 2025 study found reported GraphRAG gains partly stem from LLM-judge position, length and verbosity biases that shrink or vanish when corrected.** Validate graph gains with unbiased protocols. See [[Benchmark Saturation]].

## Entity extraction is a pluggable interface

Two implementations plus a resolver: a fast [[GLiNER]]2/spaCy path for high-volume ingestion, an LLM-extraction path for high-value documents. For Hungarian prefer [[huBERT]] / [[HuSpaCy]] — GLiNER's multilingual variants do not list Hungarian.

## Related

[[Neo4j]] · [[Text2Cypher]] · [[RAPTOR Hierarchical Summarization]] · [[Query Decomposition and Multi-Hop]] · [[Qdrant]]
