---
aliases: ["Pipeline as Platform, Model as Config"]
tags: [architecture, methodology, serving]
sources: [ocr-language-models-deployment-critical-pass-2.md, ocr-vdu-complete-study.md, ocr-arxiv-github-technical-review.md, SOTA Agentic RAG Reference Architecture - A System-Design Session Brief.md, tiered-pdf-pipeline-architecture.md]
created: 2026-08-27
updated: 2026-08-27
---

# Pipeline as Platform, Model as Config

**Rule 0.5: keep the pipeline the fixed point, make the model the replaceable part.**

The durable engineering value lives in the pipeline — layout, fan-out, assembly, formatting, validation. The recognition model behind its HTTP contract is a **config value**. This inversion is what makes the roughly two-month model-release treadmill irrelevant instead of exhausting.

## The proof that it works

The architecture kept the [[glmocr SDK]] *after rejecting the model it was built for*. When "Hungarian is 80% of the corpus" disqualified [[GLM-OCR]] in a single sentence, nothing about the pipeline had to change — only a `SERVED_NAME`. That is not luck; it is the reason the earlier work insisted on swappable parts.

## The mechanism

Three properties make it hold:

1. **Everything speaks one contract.** Every model is plain [[vLLM]] behind an OpenAI-compatible API. Multi-cloud then becomes a [[LiteLLM]] route entry rather than a re-architecture — see [[EU Sovereign GPU Hosting]].
2. **Configuration is external.** The SDK maps dotted config paths to `GLMOCR_`-prefixed environment variables, so repointing at a different model is a Kubernetes ConfigMap change with zero code.
3. **An anti-corruption layer absorbs dialect drift.** A ~200-line adapter proxy speaks the pipeline's expected dialect on one side and per-region-type prompts to whatever model is behind it on the other. The SDK stays a pinned, unmodified dependency.

## The same idea downstream

The RAG architecture applies it identically: keep every component behind a clean interface (loader, chunker, embedder, retriever, reranker, NER, graph store, web search) so the future PDF→Markdown/OCR pipeline and any model swap are drop-in, and feature-flag every extension point so the fast path ships first.

[[Tiered Page Routing]] states it concretely: *Tier 3 hides behind an OpenAI-compatible vLLM endpoint, so swapping the VLM is a config change (URL + model name), not a code change.*

## The cost you must pay for it

Three things version independently now — pipeline, adapter, and weights — and **any of them can silently break the output contract**. The insurance is region-level CI fixtures run against the live pair:

1. a **table** region → parseable HTML
2. a **formula** region → bare LaTeX
3. a **text** region containing **ő/ű**

## The end state

Combine the adapter with a **format-native fine-tune** — train the model on targets in the pipeline's expected output conventions — and the adapter's normalization shrinks toward identity. One training run buys both the language and the output contract. See [[LoRA Fine-Tuning for OCR]].

## Related

[[glmocr SDK]] · [[Golden Set and Eval Harness]] · [[LiteLLM]] · [[Tiered Page Routing]] · [[Qwen Model Family]]
