---
aliases: ["How to Query This Wiki"]
tags: [synthesis, methodology, navigation, meta, decision]
sources: [ocr-vdu-complete-study.md, tiered-pdf-pipeline-architecture.md, ocr-arxiv-github-technical-review.md, SOTA Agentic RAG Reference Architecture - A System-Design Session Brief.md, The Everything Else Map of SOTA RAG Optimization (2024–2026) - A Decision-Ready Survey for a Greenfield Agentic RAG System.md]
created: 2026-08-27
updated: 2026-08-27
---

# How to Query This Wiki

A map of what the accumulated 120 pages can answer, what they answer *well*, and — the more useful half — **what they structurally cannot answer no matter how they are asked.**

This page is navigation, not new research. Every claim below is sourced from the page it points at.

## The shape of the corpus

Fourteen sources, two halves, one join:

- **The OCR and document-parsing half** (sources 1–10) — models, serving, AKS, rasterization, fine-tuning. Ends at a markdown file.
- **The agentic RAG half** (sources 11–14) — retrieval, chunking, routing, groundedness. Begins with a slot labelled "the OCR pipeline".
- **The join** — [[The OCR-to-RAG Seam]], which no source specifies and which matters more than either half's internals.

Three constraints run through everything and give the sharpest questions their edge: **Hungarian**, **permissive licensing only**, **self-hosted**.

## Question types that get a strong answer

### 1. Decision questions — the wiki's strongest mode

- *Which recognition model, if Hungarian ő/ű accuracy is the gate?* → [[Hungarian Model Decision Matrix]]
- *Why does no single model win?* → the load-bearing finding: the only model with **verified** Hungarian ([[PaddleOCR-VL]]) is the one that **cannot be fine-tuned**, so its accuracy is a fixed ceiling.
- *What order should the probes run in?* → the matrix's probe order is sorted by information gained per hour, not by expected win.

### 2. Trade-off and comparison questions

- *What does speed cost in independent evidence?* → the two are **anti-correlated**; nothing in the field is both fast and proven for Hungarian.
- *Two-stage or end-to-end?* → [[Two-Stage vs End-to-End Document Parsing]] — the fork that decides request shape and ops surface.
- *When is [[ColPali]] worth it?* → a concrete threshold, decided at the seam: if OCR output already answers >~95% of table/chart questions, defer it.

### 3. Failure-mode questions — what breaks in production

- *The #1 production failure?* → [[Repetition Loops in VLM OCR]], with four control patterns.
- *When does a model silently correct what it can plainly see?* → [[Linguistic Crutch and Faithfulness]]; [[HunyuanOCR]] is the measured outlier.
- *What dies at the OCR-to-text handoff?* → [[The OCR-to-Text Boundary Limit]] — a measured cliff, CharXiv **0.0** for two-stage OCR plus a text LLM against **81.8–94.0** end-to-end.

### 4. Cost and sizing questions

- *Dollars per 1,000 pages?* → [[Cost per Page Model]]
- *The dominant cost lever?* → tiering, with a 5× swing — [[Tiered Page Routing]]
- *How many concurrent requests per GPU?* → the 1.25× in-flight rule in [[vLLM Continuous Batching and Concurrency Sizing]]

### 5. Architecture questions where the wiki holds an opinion

- *What must cross from OCR into retrieval?* → `source_tier`, `confidence`, `bbox` + `page_index`, `type` — and **the default failure is that all four are dropped at the markdown boundary** ([[The OCR-to-RAG Seam]]).
- *The biggest and cheapest agentic win?* → [[Adaptive RAG Routing]]
- *The highest-ROI ingestion technique?* → [[Contextual Retrieval]], which needs document context the OCR side is the only thing that has.

### 6. Licence and sovereignty questions

- *What does a permissive-only rule actually eliminate?* → [[Permissive Licensing Constraints]]: [[Provence]] (CC-BY-NC), [[Marker and Chandra]] (OpenRAIL-M revenue caps), [[HunyuanOCR]] (NOASSERTION).
- *Is an EU-located GPU enough?* → No. [[EU Sovereign GPU Hosting]] — the CLOUD Act follows **incorporation, not geography**.

### 7. Contradiction questions

`wiki/log.md` records five conflicts between sources deliberately, rather than resolving them by picking a winner — PaddleOCR-VL fine-tuning support, Hungarian in PaddleOCR-VL, the GLM-OCR language count, the Qwen3-VL OCR language count, and [[BGE-M3]] versus [[Qwen3-Embedding]] as the primary embedder. Asking *"where do my sources disagree?"* is a legitimate and productive query.

## What the wiki cannot answer

The more important half. Per [[Open Inputs and Corpus Profile]], **it is not the research that is missing — it is five facts about your own corpus that only you have**: volume, born-digital versus scanned split, page-length distribution, latency class, and data residency.

An external gap is a search away. A corpus gap **cannot be closed by more reading.** Until these are answered, every decision in the wiki is a *recommendation, not a conclusion*.

So do not query the wiki for these. **Answer them to the wiki**, then re-ask the cost and sizing questions — they become arithmetic instead of a rate with no quantity.

## The three highest-value next actions

1. **Verify that Qwen3.5 still emits `data-bbox`** — about an hour, and two of the four fields [[The OCR-to-RAG Seam]] requires depend on the answer. This replaced the Qwen3-VL Figure 2 read, which was done on 2026-08-27 and eliminated a branch: Hungarian is not among Qwen3-VL's 32 above-threshold OCR languages.
2. **Build the golden set** — 200–500 pages of your own documents, hand-transcribed, diacritic-specific error rate broken out from overall CER. [[Golden Set and Eval Harness]] is Rule 0, and it is simultaneously the RAG side's eval set ([[RAG Evaluation]]). One artifact, both halves.
3. **Close the eval-harness implementation gap** — the lint pass left it open on purpose, because no ingested source contains it. [[Open Inputs and Corpus Profile]] names it as the gap most likely to stall execution.

## Related

[[Open Inputs and Corpus Profile]] · [[Hungarian Model Decision Matrix]] · [[The OCR-to-RAG Seam]] · [[Cost per Page Model]] · [[Golden Set and Eval Harness]] · [[Pipeline as Platform, Model as Config]]
