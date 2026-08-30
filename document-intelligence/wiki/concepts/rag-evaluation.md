---
aliases: ["RAG Evaluation"]
tags: [rag, evaluation, methodology, observability]
sources: [SOTA Agentic RAG Reference Architecture - A System-Design Session Brief.md, Query Modification for a SOTA Agentic RAG System - Taxonomy Benchmarks and a Staged Design.md, The Everything Else Map of SOTA RAG Optimization (2024–2026) - A Decision-Ready Survey for a Greenfield Agentic RAG System.md]
created: 2026-08-27
updated: 2026-08-27
---

# RAG Evaluation

The downstream twin of the [[Golden Set and Eval Harness]] discipline: **build a golden query set from real user queries early, and gate every change against it.**

## The tools

| Tool | Role |
|---|---|
| **RAGAS** | reference-free faithfulness, answer relevancy, context precision, context recall — best for fast iteration |
| **DeepEval** | pytest-style, CI/CD-native, broadest metric library including agentic metrics — best for *guarding* a pipeline |
| **TruLens** | production monitoring and tracing (OpenTelemetry) |
| **ARES** (Saad-Falcon et al., NAACL 2024) | automated eval with fine-tuned judges |

**The loop:** start with RAGAS for a baseline → add DeepEval tests in CI so every retriever or prompt change is gated → add TruLens once real traffic flows.

## Measure retrieval separately from generation

Track **nDCG@k, Recall@k, MRR** for retrieval and faithfulness/correctness for generation. A better-sounding answer can be grounded in worse evidence.

**Always track latency and per-query cost alongside quality.** A pipeline that scores higher on faithfulness but takes 8 s and 3 extra LLM calls may be worse in production.

## A/B each transformation in isolation

Hold the rest of the pipeline fixed. **Rewrite plausibility is a poor proxy for retrieval lift.** Report Recall@k and nDCG@10 plus end-to-end faithfulness, break results down per language (HU/EN), and **feature-flag every transformation** so a regression can be switched off rather than reverted.

**Use confidence intervals at realistic sample sizes** — the RAG-Fusion harness shows many "gains" vanish under proper CIs.

## Do not trust a single automated score

Independent work (**GroUSE**, arXiv 2409.06595) shows LLM-judge faithfulness scorers have blind spots. A 2025 study found reported GraphRAG gains partly stem from **LLM-judge position, length and verbosity biases** that shrink or vanish when corrected. See [[Benchmark Saturation]].

## Why public benchmarks will not save you here

Hungarian MTEB coverage is **classification-only — no retrieval tasks**, MIRACL has **no Hungarian split**, and there is **no public Hungarian embedding leaderboard or Hungarian-specific reranker**. Benchmark transfer is also weak in general: HotpotQA/MuSiQue/2Wiki are Wikipedia-style open-domain QA, and a document-intelligence corpus behaves differently.

**The eval harness is non-optional**, and it is the only thing that resolves the open questions the surveys deliberately leave open — [[Qwen3-Embedding]] versus [[BGE-M3]] on Hungarian, miniCOIL versus BM25, whether [[ColPali]] earns its storage, whether the agentic branch earns its latency.

## Observability

Instrument with **OpenTelemetry GenAI semantic conventions**; use **Langfuse** or **Arize Phoenix** for tracing and eval. Track token budgets per stage and use small-model cascades (cheap model first, escalate on low confidence) — which aligns with the [[Adaptive RAG Routing]] router you already have. Adopt from day one.

## Related

[[Golden Set and Eval Harness]] · [[Benchmark Saturation]] · [[Adaptive RAG Routing]] · [[Hallucination Detection in RAG]]
