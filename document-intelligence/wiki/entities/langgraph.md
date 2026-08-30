---
aliases: ["LangGraph"]
tags: [rag, orchestration, agentic, framework]
sources: [SOTA Agentic RAG Reference Architecture - A System-Design Session Brief.md, Query Modification for a SOTA Agentic RAG System - Taxonomy Benchmarks and a Staged Design.md, The Everything Else Map of SOTA RAG Optimization (2024–2026) - A Decision-Ready Survey for a Greenfield Agentic RAG System.md, Deep-Dive - 10 RAG Optimization Techniques for a Self-Hosted Agentic Stack.md]
created: 2026-08-27
updated: 2026-08-27
---

# LangGraph

MIT. The recommended agentic control plane: an explicit stateful graph with clear decision and branch points, **bounded loops**, and LangSmith observability. Since LangChain 1.0 (Oct 2025) it is the execution engine LangChain agents run on.

It is chosen specifically because the architecture's hard parts are *routing and bounded self-correction* — see [[Adaptive RAG Routing]] and [[Corrective RAG]] — and LangGraph's state model removes the boilerplate those need.

## What becomes a node

| Node | Concept |
|---|---|
| complexity router | [[Adaptive RAG Routing]] |
| conversational rewrite, Self-Query extraction | [[Query Rewriting and Expansion]] |
| decomposition / step-back / IRCoT loop | [[Query Decomposition and Multi-Hop]] |
| hybrid retrieve + RRF + rerank | [[Hybrid Retrieval and RRF]] |
| Cypher tool calls | [[GraphRAG and Document Graphs]] |
| CRAG relevance grader, web fallback | [[Corrective RAG]] |
| context pruning + ordering | [[Context Pruning and Lost-in-the-Middle]] |
| groundedness gate | [[Hallucination Detection in RAG]] |
| tool capability control | [[Indirect Prompt Injection]] |

## Alternatives

| Framework | License | Strength |
|---|---|---|
| **LangGraph** | MIT | stateful graphs, bounded loops, routing — recommended |
| [[LlamaIndex]] | MIT | strongest for ingestion/indexing/retrieval; many teams pair LlamaIndex retrieval + LangGraph orchestration |
| Haystack (deepset) | Apache-2.0 | clean typed components, enterprise focus (RBAC, monitoring), native web-search fallback |
| Custom | — | viable, but more boilerplate for the routing/loop logic |

## Design rules that outrank the framework choice

- **Bound every loop.** Cap agentic iterations at ~3–4 hops; unbounded loops cost latency and amplify errors on noisy input.
- **Keep the tool count small (3–5).** Too many similar tools degrade selection accuracy.
- **Feature-flag every extension point** so the fast path ships first and the graph/NER/web-search branches are added without refactoring.

## Related

[[Adaptive RAG Routing]] · [[Corrective RAG]] · [[LlamaIndex]] · [[RAG Evaluation]]
