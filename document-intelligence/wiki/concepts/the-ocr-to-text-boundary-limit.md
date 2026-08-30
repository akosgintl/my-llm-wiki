---
aliases: ["The OCR-to-Text Boundary Limit"]
tags: [architecture, failure-modes, kie, vlm]
sources: [ocr-language-models-deployment-critical-pass-2.md, ocr-vdu-complete-study.md, ocr-arxiv-github-technical-review.md, The Everything Else Map of SOTA RAG Optimization (2024–2026) - A Decision-Ready Survey for a Greenfield Agentic RAG System.md]
created: 2026-08-27
updated: 2026-08-27
---

# The OCR-to-Text Boundary Limit

**Anything visual dies at the OCR→text boundary.** This is measured, not theoretical.

## The number

Qianfan's comparison (arXiv 2603.13398, Table 6): a two-stage OCR + Qwen3-4B system scored **CharXiv 0.0 against 81.8–94.0 for an end-to-end VLM**. Not degraded — **zero**. Visual QA collapses generally in the same setup.

## What survives and what does not

| Survives as text | Dies at the boundary |
|---|---|
| field semantics (names, amounts, dates) | charts and plots |
| document structure expressed in markdown | checkboxes |
| tables that linearize cleanly | stamps and seals |
| | spatial layout relationships |

## The two legitimate slots for a text-only LLM

A text-only model **cannot** fill the recognition slot — no vision encoder, and the pipeline sends pixels. But downstream it has real work:

**1. KIE over OCR'd markdown.** This matches the recommended output contract (markdown canonical + a separate extraction stage), and broad-language text LLMs handle Hungarian field semantics well.

**2. Post-OCR diacritic restoration.** Tempting — *kör* versus *kőr* genuinely is a language-model problem — and **dangerous unrestrained**: it will fluently "fix" names, amounts and IDs. If used: diacritic-only edit constraint, edit-distance guard, and **never on numeric or ID fields**. See [[Hungarian OCR and the Double Acute Test]].

## The routing rule

**Route page-level visual questions to a VLM, or do not answer them.** Do not let a text pipeline produce a confident answer about a chart it never saw.

## The RAG-side corollary

This limit is precisely why **[[ColPali]] vision retrieval** is the biggest untapped opportunity for a pipeline that already renders page images: late-interaction retrieval over page images **bypasses the boundary entirely** for tables, charts and scans.

And it gives the adoption threshold a concrete form: **if OCR quality already yields >~95% correct answers on table/chart questions, defer ColPali.** Its value is exactly proportional to how often you are hitting this limit.

**And the limit is moving.** Checked 2026-08-27: Qwen3.5-4B scores **70.8 on CharXiv (RQ)** and 9B scores **73.0**, against 56.6 for Qwen3-VL-30B — a +16.4 point generational jump on the very benchmark this page is built on ([[Qwen Model Family]]). That does not repeal the 0.0-vs-81.8 result, which is about the *two-stage* architecture and still holds. What it changes is the cheapest way around it: a single small end-to-end VLM now recovers a meaningful share of what dies at the boundary, without standing up a second retrieval stack. **Measure the end-to-end model on your own chart pages before committing to [[ColPali]]** — the two are alternatives for the same failure, and one of them is already in the pipeline.

Where this limit gets decided in the pipeline: [[The OCR-to-RAG Seam]].

## Related

[[ColPali]] · [[Two-Stage vs End-to-End Document Parsing]] · [[Confidence Engineering]] · [[Hungarian OCR and the Double Acute Test]]
