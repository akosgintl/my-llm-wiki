---
aliases: ["Anthropic"]
tags: [organization, rag]
sources: [The Everything Else Map of SOTA RAG Optimization (2024–2026) - A Decision-Ready Survey for a Greenfield Agentic RAG System.md, Deep-Dive - 10 RAG Optimization Techniques for a Self-Hosted Agentic Stack.md]
created: 2026-08-27
updated: 2026-08-27
---

# Anthropic

Present in this corpus not as a model vendor but as the source of two RAG design results that the self-hosted architecture reimplements without their models.

## Contextual Retrieval (Sept 2024)

The single highest-ROI ingestion technique in the survey — prepend an LLM-generated, chunk-specific context blurb before embedding and before BM25 indexing. Their published failure-rate reductions (35% / 49% / 67%) are the headline numbers of [[Contextual Retrieval]], and their published prompt is reused verbatim.

Their prompt-caching economics ($1.02 per million document tokens) translate directly to a self-hosted setup: [[vLLM]]'s automatic prefix caching gives the same effect when the whole document is the shared prefix.

Worth noting they also published the **negative** result: generic document-summary prepending gave "very limited gains" and summary-based indexing "low performance." The win is specifically chunk-level context.

## The long-context guidance

Their stated rule — *if your knowledge base is under ~200k tokens (~500 pages), skip RAG entirely and prompt-cache the corpus* — is why the reference architecture keeps a "small-corpus / whole-document stuffing" fast path gated by the [[Adaptive RAG Routing]] router, rather than treating RAG as universal.

They also found top-20 chunks outperformed top-10 and top-5 in their setup, while cautioning that more chunks eventually distract — tune it.

## Also

The **Citations API pattern** (stable chunk IDs, inline citations, quote-first generation) is emulated self-hosted and verified post-hoc with [[LettuceDetect]].

## Caveat

All of these numbers come from Anthropic's own internal evaluation, using 1−recall@20 with Gemini Text 004 embeddings and a Cohere reranker — **not your stack**. Treat as directional and re-run locally.

## Related

[[Contextual Retrieval]] · [[Adaptive RAG Routing]] · [[Context Pruning and Lost-in-the-Middle]]
