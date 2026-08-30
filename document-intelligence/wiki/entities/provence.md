---
aliases: ["Provence"]
tags: [rag, context-pruning, reranking, licensing, model]
sources: [The Everything Else Map of SOTA RAG Optimization (2024–2026) - A Decision-Ready Survey for a Greenfield Agentic RAG System.md, Deep-Dive - 10 RAG Optimization Techniques for a Self-Hosted Agentic Stack.md]
created: 2026-08-27
updated: 2026-08-27
---

# Provence

Naver's DeBERTa-v3-based context pruner (ICLR 2025, arXiv 2501.16214). It **unifies reranking and sentence-level context pruning** in one cross-encoder:

- a **pruning head** — token-level binary mask, thresholded ~0.1–0.5
- a **reranking head** — BOS-token relevance score, MSE-distilled from a DeBERTa reranker

It cross-encodes query + passage and dynamically decides *how much* to prune per context, robustly across domains. Removes ~49% of context with minimal quality loss and negligible latency, because it is an encoder rather than an LLM. Trained on MS MARCO (3.2M docs) + Natural Questions with a SPLADE-v3 + DeBERTa-v3 pipeline.

## The licensing flag — this is the important part

The official weights `naver/provence-reranker-debertav3-v1` are **CC-BY-NC-4.0 (non-commercial)** per the HF blog and independent analyses. One HF README snapshot shows `license: cc-by-4.0`, but the model card's License section and Naver's own blog state CC-BY-NC-4.0.

**Treat as non-commercial until Naver clarifies. Do not deploy the weights commercially.** This is one of only two licensing risks in the ten-technique shortlist (the other being [[ColPali]]'s backbone). See [[Permissive Licensing Constraints]].

## Permissive alternatives

The technique is still worth adopting — just not these weights:

1. **Train your own extractive pruner** on a permissive DeBERTa or ModernBERT. The paper shows the standalone pruner is a simple BERT + pruning head trained on LLM-generated targets.
2. **RECOMP-style extractive pruning.**
3. **[[bge-reranker-v2-m3]] (Apache-2.0) for reranking + a permissive sentence-level relevance classifier for pruning.**

## Versus LLMLingua

LongLLMLingua boosts NaturalQuestions by up to 21.4% at ~4× compression and reports up to 94% cost reduction on LooGLE — but it is question-aware (so no context caching), doubles compute versus LLMLingua, and is the slowest pruner available. **Evaluate only** if you need extreme token compression and can absorb the latency.

## Related

[[Context Pruning and Lost-in-the-Middle]] · [[bge-reranker-v2-m3]] · [[Permissive Licensing Constraints]]
