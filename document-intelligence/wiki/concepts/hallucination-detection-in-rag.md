---
aliases: ["Hallucination Detection in RAG"]
tags: [rag, hallucination, reliability, groundedness]
sources: [The Everything Else Map of SOTA RAG Optimization (2024–2026) - A Decision-Ready Survey for a Greenfield Agentic RAG System.md, Deep-Dive - 10 RAG Optimization Techniques for a Self-Hosted Agentic Stack.md, SOTA Agentic RAG Reference Architecture - A System-Design Session Brief.md]
created: 2026-08-27
updated: 2026-08-27
---

# Hallucination Detection in RAG

An **inline groundedness gate** between generation and delivery. Non-negotiable for a bilingual, possibly-untrusted corpus — it belongs in v1, not v2.

## The gate

Run a detector over (retrieved context, question, draft answer). If unsupported spans exceed a threshold, branch to:

1. **regenerate** constrained to supported content,
2. **re-retrieve**, or
3. **abstain and flag**.

[[LettuceDetect]] is the recommended detector: MIT, ModernBERT token-level, **79.22% F1 on RAGTruth** (versus GPT-4 prompt-based at 63.4%), ~30× smaller than the best LLM judges, 30–60 samples/s on one GPU, and its v2 line covers **Hungarian** via the 14-language PsiloQA set. Because it returns **token spans**, you can surface exactly which text is unsupported, or force citations for the flagged spans.

Alternatives: MiniCheck (MiniCheck-FT5 770M at GPT-4-level for 400× less cost — English), Lynx, Vectara HHEM-2.1 (English/French/German, no Hungarian).

## Where it sits relative to the other checks

Three different things get checked at three different points, and they are not substitutes:

| Check | What it inspects | Concept |
|---|---|---|
| CRAG relevance grader | the **retrieved context**, before generation | [[Corrective RAG]] |
| Self-RAG reflection | whether to retrieve, and self-critique | [[Corrective RAG]] |
| **Groundedness gate** | the **draft answer** against the context | this page |
| Citation verification | that cited chunk IDs actually support the claims | [[Context Pruning and Lost-in-the-Middle]] |

Self-RAG is worth noting on its own: a 2025 MDPI *Electronics* study of 12 RAG variants reported it had the **lowest hallucination rate (5.8%)** among them.

## Scope it honestly

An independent 2026 benchmark found LettuceDetect-large **weak on code-agent hallucinations (0.17 span-F1)** — scope it to prose and document QA. It is trained primarily on the RAGTruth distribution, and its Hungarian performance should be validated locally.

More generally: **do not treat a single automated score as ground truth.** Independent work (GroUSE, arXiv 2409.06595) shows LLM-judge faithfulness scorers have blind spots. See [[Benchmark Saturation]].

## The parallel on the OCR side

The document pipeline faces the same problem one layer earlier and solves it the same way — an engineered ensemble rather than a model feature: layout scores + logprobs + value-in-OCR grounding + deterministic validators, calibrated so a threshold routes to human review. See [[Confidence Engineering]].

## Related

[[LettuceDetect]] · [[Corrective RAG]] · [[Confidence Engineering]] · [[RAG Evaluation]]
