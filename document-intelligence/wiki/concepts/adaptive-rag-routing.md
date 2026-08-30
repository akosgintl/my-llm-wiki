---
aliases: ["Adaptive RAG Routing"]
tags: [rag, agentic, routing, cost, architecture]
sources: [SOTA Agentic RAG Reference Architecture - A System-Design Session Brief.md, Query Modification for a SOTA Agentic RAG System - Taxonomy Benchmarks and a Staged Design.md, The Everything Else Map of SOTA RAG Optimization (2024–2026) - A Decision-Ready Survey for a Greenfield Agentic RAG System.md]
created: 2026-08-27
updated: 2026-08-27
---

# Adaptive RAG Routing

**The biggest single win in agentic RAG, and the cheapest.** Classify query complexity, then send easy queries down a single-pass fast path and escalate only genuinely complex ones into an agentic loop.

## The canonical pattern

**Adaptive-RAG** (Jeong, Baek, Cho, Hwang & Park, NAACL 2024, pp. 7036–7050, DOI 10.18653/v1/2024.naacl-long.389): a T5-Large classifier pre-assesses complexity and routes among (a) no retrieval, (b) single-step retrieval, (c) multi-step/iterative retrieval — and *"can match always-expensive baselines with substantially lower cost."*

**Lightweight routers suffice.** The RAGRouter-Bench baseline study (arXiv 2604.03455) reports that **lexical TF-IDF features "outperform semantic sentence embeddings by 3.1 macro-F1 points"**, with a best classifier around ~92% accuracy / ~0.92 macro-F1 and roughly a quarter of tokens saved versus always taking the most expensive path. (Confirm exact figures against its Table 2 before quoting.) In practice a keyword heuristic or a small classifier is enough to start.

## Why routing rather than always-agentic

**Multi-hop is a categorical capability gap on compositional questions — and dead weight otherwise.**

| Benchmark | NaiveRAG | Iterative / decomposition |
|---|---|---|
| MuSiQue (2–4 hop) | ~8.2 EM / 47 Recall@10 | **~39–44 EM / 75–90 Recall@10** |
| HotpotQA, 2WikiMultiHopQA | smaller gap — often answerable in fewer hops | |

But the cost is real: structured iterative methods run **~2–3× slower** (~8.1 s vs ~3.3 s on HotpotQA). And on single-fact lookups the gains vanish entirely — WildGraphBench (2026) found *"flat baselines like NaiveRAG remain competitive on single-fact retrieval"* and that GraphRAG *"can be more expensive than NaiveRAG or BM25 without clear gains."*

**A robustness caveat that argues for bounding the loop:** on noisy or ASR-corrupted input, multi-hop extensions can **amplify** errors (F1 gap 36–67% larger under entity-graph linking + iterative reformulation).

## Where the router sits

**After** a cheap query-understanding stage, so it classifies a clean, standalone query — normalization, language detection, conversational rewrite and metadata extraction all run *pre-router*. Expensive transformations run *post-router*, on the agentic branch only. See [[Query Rewriting and Expansion]].

```
Query → [Stage 0: normalize, rewrite, Self-Query] → ROUTER
                    simple ──→ FAST PATH:  hybrid → RRF → rerank → answer
                   complex ──→ AGENTIC:    decompose → per-hop retrieve → self-critique
```

## What else it gates

Once the router exists it becomes the general escalation mechanism:

- routing hard queries to an expensive LLM reranker — see [[Two-Stage Retrieve-Then-Rerank]]
- a "small corpus / whole-document stuffing" fast path for tiny document sets
- small-model cascades: cheap model first, escalate on low confidence

## Adoption threshold

Ship the fast path first. **Build the agentic branch when >20–30% of queries return low-faithfulness answers, or your query logs show multi-entity/multi-hop questions** — and adopt it only if it improves EM/F1 on your multi-hop eval slice enough to justify the 2–3× latency. If not, keep it gated to a narrow query class.

## Related

[[Query Decomposition and Multi-Hop]] · [[Corrective RAG]] · [[LangGraph]] · [[Two-Stage Retrieve-Then-Rerank]]
