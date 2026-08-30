---
aliases: ["olmOCR"]
tags: [ocr, vlm, document-parsing, benchmarks, model]
sources: [Open-Source OCR per Visual Document Understanding Models for Self-Hosted AKS Pipelines - A Comparative Investigation (August 2026).md, ocr-arxiv-github-technical-review.md, Deep Technical Review - Open-Source OCR - Visual Document Understanding Models - Missed Details and Pipeline Implications.md, Local PDF-Markdown and Document Parsing - State of the Art (August 2026).md, tiered-pdf-pipeline-architecture.md]
created: 2026-08-27
updated: 2026-08-27
---

# olmOCR

[[Allen Institute for AI]]'s fully-open English PDF linearization system (arXiv 2510.19817). `olmOCR-2-7B-1025` is built on Qwen2.5-VL-7B and fine-tuned on olmOCR-mix-1025 (270k pages + 20k added hard handwritten/typewritten pages). Apache-2.0, with weights, data, training and inference code all released.

## RLVR — the third reward paradigm

Trained with **RLVR via GRPO using binary unit-test rewards** aggregated to a page-level pass rate (e.g. 4 of 6 = 0.67). A synthetic pipeline renders any document to clean HTML, then extracts verifiable unit tests from it; math-heavy arXiv pages supply the hard cases; claude-sonnet was the teacher for HTML generation.

This matters beyond olmOCR itself. Alongside [[GLM-OCR]]'s GRPO metric-rewards and plain SFT, RLVR is a third objective design — and it **turns CI fixtures into training rewards**. Any pipeline that already runs region-level fixtures (table→parseable HTML, formula→bare LaTeX, text containing ő/ű) can reuse them as an RL signal. See [[LoRA Fine-Tuning for OCR]].

## Pipeline heuristics worth stealing

Automatic page retries, dynamic temperature scaling on retry, quality filters, and perplexity-style rejection — a productionized version of the hedging and loop-detector layer described in [[Document Fan-Out and Fan-In]].

## Numbers

- **[[olmOCR-Bench]] 82.4** (+14.2 over the prior version), beating marker (76.1) and [[MinerU]] (75.8) at release. Note the benchmark is olmOCR's own — read its placement of competitors accordingly.
- [[OCR Arena]] #19 (ELO 1382), beating [[GLM-OCR]] with a 92.3% win rate across 13 matchups.
- ~1.78 pages/s; 7B, wants H100-class hardware. Excellent S3/batch pipeline for large-scale linearization.

## Why it is not a Hungarian candidate

**English-only by design.** It is explicitly excluded from Hungarian-first architectures on that basis — see [[Tiered Page Routing]]. Its ideas (RLVR-from-fixtures, retry heuristics) belong in the pipeline regardless of which model wins.

## Related

[[olmOCR-Bench]] · [[Marker and Chandra]] · [[Allen Institute for AI]] · [[Qwen Model Family]]
