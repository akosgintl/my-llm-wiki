---
aliases: ["Two-Stage vs End-to-End Document Parsing"]
tags: [architecture, ocr, document-parsing]
sources: [Open-Source OCR per Visual Document Understanding Models for Self-Hosted AKS Pipelines - A Comparative Investigation (August 2026).md, ocr-arxiv-github-technical-review.md, ocr-vdu-complete-study.md, Deep Technical Review - Open-Source OCR - Visual Document Understanding Models - Missed Details and Pipeline Implications.md]
created: 2026-08-27
updated: 2026-08-27
---

# Two-Stage vs End-to-End Document Parsing

The field's central architectural fork, and it determines far more downstream than accuracy does.

## Three shapes, not two

| Shape | Models | Layout decision? |
|---|---|---|
| **Separate detector + VLM** | [[GLM-OCR]], [[PaddleOCR-VL]] (both via [[PP-DocLayout-V3]]) | Yes — see [[Layout Stage Economics]] |
| **Two stages inside one model** | [[MinerU]] (1.2B): layout on a *downsampled* global image, recognition on *native-resolution crops* | **No** — nothing to place |
| **End-to-end page model** | [[DeepSeek-OCR]], [[Baidu Unlimited-OCR]], [[dots.ocr]], [[HunyuanOCR]] | No |

MinerU's coarse-to-fine trick is the interesting middle: it dodges the O(N²) visual-token blowup of native-resolution end-to-end models without needing a separate detector or a placement decision.

## What the choice actually decides

**Request shape.** Two-stage models take **one region crop** per request, so the pipeline fans out N regions × M pages. End-to-end models take **one full page**. See [[Document Fan-Out and Fan-In]].

**Cost per request.** Region crops become very few tokens through GLM-OCR's downsampling connector — which is exactly why region-level requests are so cheap and why high turnover feeds the batch aggressively.

**Operational surface.** Two-stage adds a CPU tier to size, a second failure mode (layout errors propagating into recognition), and a hard requirement to run the *full* pipeline — serving [[PaddleOCR-VL]]'s bare VLM increases hallucination and does not reproduce the paper. End-to-end has none of that and no bounding boxes either.

**Reading order.** The detector emits it (Global-Pointer pointer mechanism), so the assembly step should consume it rather than re-sorting by y-coordinate. End-to-end models learn it internally — [[DeepSeek-OCR]] 2's DeepEncoder V2 exists specifically to reorder visual tokens into human reading order, improving reading-order edit distance 0.085 → 0.057.

## Robustness: the evidence favors two-stage

The [[Linguistic Crutch and Faithfulness]] critique found that **"traditional pipeline OCR methods are significantly more robust than end-to-end methods"** under semantic corruption, across 13 baselines. End-to-end models trade robustness for simplicity.

## Do not chain OCR + text-LLM and call it two-stage

A genuinely different failure: routing a page through OCR to text and then asking a text LLM about it destroys everything visual. Qianfan's comparison measured **CharXiv 0.0 for two-stage OCR+Qwen3-4B versus 81.8–94.0 end-to-end**. See [[The OCR-to-Text Boundary Limit]].

## Related

[[Layout Stage Economics]] · [[Document Fan-Out and Fan-In]] · [[The OCR-to-Text Boundary Limit]] · [[Tiered Page Routing]]
