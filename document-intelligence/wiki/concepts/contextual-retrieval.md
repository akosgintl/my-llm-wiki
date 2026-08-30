---
aliases: ["Contextual Retrieval"]
tags: [rag, ingestion, retrieval, optimization]
sources: [The Everything Else Map of SOTA RAG Optimization (2024–2026) - A Decision-Ready Survey for a Greenfield Agentic RAG System.md, Deep-Dive - 10 RAG Optimization Techniques for a Self-Hosted Agentic Stack.md]
created: 2026-08-27
updated: 2026-08-27
---

# Contextual Retrieval

**The highest single-ROI addition to a RAG pipeline.** Prepend an LLM-generated, chunk-specific context blurb (50–100 tokens) to each chunk **before embedding and before building the BM25 index**.

It attacks the root cause of RAG failure — context-poor chunks — at ingestion, so it compounds with everything downstream. See [[Anthropic]].

## The numbers

Metric = 1−recall@20:

| Configuration | Top-20 retrieval failure rate |
|---|---|
| Baseline | 5.7% |
| + Contextual **Embeddings** | 3.7% (**−35%**) |
| + Contextual **BM25** | 2.9% (**−49%**) |
| + Reranking | 1.9% (**−67%**) |

Note the second row: **half the gain comes from contextualizing the BM25 index too.** Do not index the blurb only on the dense side.

## The prompt (published verbatim)

```
<document>
{{WHOLE_DOCUMENT}}
</document>
Here is the chunk we want to situate within the whole document
<chunk>
{{CHUNK_CONTENT}}
</chunk>
Please give a short succinct context to situate this chunk within the overall
document for the purposes of improving search retrieval of the chunk.
Answer only with the succinct context and nothing else.
```

Full pipeline: chunk (a few hundred tokens) → contextualize → dual-index (dense + BM25) → retrieve top-150 by RRF → rerank → top-20 into the prompt.

## Self-hosted economics

The economics hinge on **prompt caching**: you load the document once and reference the cached content per chunk. Anthropic's figure is $1.02 per million document tokens.

Self-hosted, [[vLLM]]'s **automatic prefix caching gives the equivalent** — put the whole document as a shared prefix so per-chunk generation reuses the document KV cache. Run it with a small instruct model (Qwen3-4B/8B-Instruct or Llama-3.1-8B) on vLLM, then dual-write to [[Qdrant]] and [[OpenSearch]].

For Hungarian, write the contextualizer instruction in the document's language or bilingually, and validate output quality.

## The negative result that defines the technique

Anthropic found generic **document-summary prepending** gave *"very limited gains"* and summary-based indexing *"low performance."* The win is specifically **chunk-level** context. Do not substitute a cheaper summary.

## Limits

- Costs scale with corpus churn — every re-chunk re-contextualizes.
- Practitioners report the LLM sometimes emits boilerplate ("This chunk discusses…") that adds noise. **Constrain the output and evaluate it.**
- The headline numbers are **Anthropic's own internal eval** using Gemini Text 004 embeddings + a Cohere reranker — not your stack. Treat as directional; re-run locally on 1−recall@20.

## Cheaper complement: late chunking

**Late Chunking** (Jina) embeds the *whole* document through a long-context model, then mean-pools token embeddings per chunk — preserving pronouns, references and thematic context with **no LLM calls**. It requires a long-context embedder ([[BGE-M3]] at 8k, [[Qwen3-Embedding]] at 32k both qualify), but only captures *within-document* context.

**Pair them**: late chunking for cheap within-document context, contextual retrieval for cross-document facts, where budget allows.

Its document context should come from the OCR block tree rather than be re-derived — see [[The OCR-to-RAG Seam]].

## Related

[[Chunking Strategies]] · [[Hybrid Retrieval and RRF]] · [[Anthropic]] · [[Qwen3-Embedding]]
