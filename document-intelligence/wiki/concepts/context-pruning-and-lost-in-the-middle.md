---
aliases: ["Context Pruning and Lost-in-the-Middle"]
tags: [rag, generation, context, optimization]
sources: [The Everything Else Map of SOTA RAG Optimization (2024–2026) - A Decision-Ready Survey for a Greenfield Agentic RAG System.md, Deep-Dive - 10 RAG Optimization Techniques for a Self-Hosted Agentic Stack.md]
created: 2026-08-27
updated: 2026-08-27
---

# Context Pruning and Lost-in-the-Middle

What happens between reranking and generation. Cheap, and mostly free.

## Lost-in-the-middle

Liu et al., arXiv 2307.03172, TACL 2024: **LLMs use information best at the beginning or end of the context**, and performance degrades significantly for relevant information in the middle — **even in long-context models**.

Mitigations, in order of cost:

1. **Reorder retrieved chunks so the most relevant are first and last.** Free — pure prompt-assembly logic. Adopt immediately.
2. **Aggressively prune** so less lands in the middle at all.
3. Attention-sorted reordering (Peysakhovich & Lerer) using models' preferential attention.
4. Query-aware contextualization (query before *and* after the documents) — helps key-value retrieval, little for multi-document QA.
5. **Keep top-k modest.** Anthropic found 20 > 10 > 5 in their setup, but more chunks eventually distract — tune it.

## Pruning

[[Provence]] (ICLR 2025) removes ~49% of context with minimal quality loss and negligible latency, because it is an encoder rather than an LLM. **But its weights are CC-BY-NC-4.0 — do not deploy them commercially.** Use a permissive alternative: train your own extractive pruner (the standalone pruner is a simple BERT + pruning head on LLM-generated targets), RECOMP-style extraction, or [[bge-reranker-v2-m3]] for reranking plus a permissive sentence-level relevance classifier.

LongLLMLingua boosts NaturalQuestions by up to 21.4% at ~4× compression and up to 94% cost reduction on LooGLE — but it is question-aware (so no context caching), doubles compute versus LLMLingua, and is the slowest pruner. **Evaluate only.**

## Formatting and citations

- **XML-style delimiters** (`<document id=..>`) tend to be parsed more reliably than markdown for context blocks — and they pair with spotlighting, so the same choice serves [[Indirect Prompt Injection]] defense.
- **Stable per-chunk IDs**, cited inline by the generator.
- **Quote-first generation** — extract supporting quotes before composing the answer.
- **Verify citations post-hoc** with [[LettuceDetect]].
- Budget the context window explicitly per stage.

## Long context is not a substitute for retrieval

On LongBench v2, Qwen2.5 and GLM-4-Plus performed **better at 32k retrieval context than with the full 128k without RAG** (Qwen2.5 +4.1%). Only GPT-4o leveraged 128k well, and still lagged its own no-RAG score (−0.6%).

The exception: [[Anthropic]]'s guidance that a knowledge base under ~200k tokens (~500 pages) can skip RAG entirely and just be prompt-cached. Keep RAG primary and add a **small-corpus fast path** gated by the [[Adaptive RAG Routing]] router.

## Related

[[Provence]] · [[Two-Stage Retrieve-Then-Rerank]] · [[Hallucination Detection in RAG]] · [[KV-Cache Reuse for RAG]]
