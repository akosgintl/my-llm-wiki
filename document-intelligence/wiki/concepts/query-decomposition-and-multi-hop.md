---
aliases: ["Query Decomposition and Multi-Hop"]
tags: [rag, agentic, retrieval, query-understanding]
sources: [Query Modification for a SOTA Agentic RAG System - Taxonomy Benchmarks and a Staged Design.md, SOTA Agentic RAG Reference Architecture - A System-Design Session Brief.md]
created: 2026-08-27
updated: 2026-08-27
---

# Query Decomposition and Multi-Hop

The techniques that belong **only on the agentic branch**, because they deliver large gains on genuinely compositional questions and actively hurt precise single-hop ones.

## Decomposition

Break a compositional question into logically ordered single-hop sub-questions, retrieve for each, aggregate.

Lineage: Self-Ask (Press et al., 2022), Decomposed Prompting (Khot et al., 2022), least-to-most, **IRCoT** (Trivedi et al., ACL 2023, arXiv 2212.10509).

**IRCoT** interleaves retrieval with each chain-of-thought step: up to **+21 retrieval points and +15 QA points** on HotpotQA / 2WikiMultihopQA / MuSiQue / IIRC — and it works with small models (Flan-T5-large).

**2025 SOTA**: "Question Decomposition for RAG" (Ammann, Golde & Akbik, ACL SRW 2025, arXiv 2507.00355) reports *"gains in retrieval (MRR@10: +36.7%) and answer accuracy (F1: +11.6%) over standard RAG baselines"* on MultiHop-RAG/HotpotQA — but warns that *"if a query is already precise, decomposition can introduce"* noise, and that *"longer contexts can degrade downstream performance."*

"Query Decomposition for RAG: Balancing Exploration-Exploitation" (arXiv 2510.18633) adds adaptive per-subquery retrieval budgets.

Prompt: *"Generate logically-ordered single-hop sub-questions needed to answer; stop at N; if the question is already single-hop, return it unchanged."*

## Step-back abstraction

Step-Back Prompting (Zheng et al., ICLR 2024, arXiv 2310.06117) elicits a higher-level abstraction question first, so retrieval can surface governing principles rather than an over-narrow match.

Gains: *"MMLU (Physics and Chemistry) by 7% and 11% respectively, TimeQA by 27%, and MuSiQue by 7%."*

**But GSM8K gains were minimal** (84.3%, "on par with zero-shot CoT") — abstraction helps *knowledge-intensive* tasks, not simple arithmetic. Enable it only for the reasoning-intensive intent class where it beats plain decomposition.

## The bounding rules

1. **Bound the hops** — ≤3–4 iterations. Unbounded loops cost latency and, on noisy input, **amplify errors** (F1 gap 36–67% larger under entity-graph linking + iterative reformulation).
2. **Only on the complex branch.** The router decides — see [[Adaptive RAG Routing]].
3. **Per-hop, do the full retrieval stack**: hybrid retrieve + RRF + rerank, optionally plus graph retrieval via curated Cypher tools.
4. **End with self-critique / a groundedness check** before answering. See [[Hallucination Detection in RAG]].

## The payoff, in context

| Benchmark | NaiveRAG | Decomposition/iterative |
|---|---|---|
| MuSiQue (2–4 hop) | ~8.2 EM / 47 Recall@10 | ~39–44 EM / 75–90 Recall@10 |
| HotpotQA / 2Wiki | smaller gap | |
| Latency | ~3.3 s | ~8.1 s (**2–3×**) |

A categorical capability gap where it applies, and pure cost where it does not. That asymmetry is the entire argument for routing.

## Related

[[Adaptive RAG Routing]] · [[Query Rewriting and Expansion]] · [[GraphRAG and Document Graphs]] · [[LangGraph]]
