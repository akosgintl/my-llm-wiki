---
aliases: ["CHAOS-Bench"]
tags: [benchmarks, evaluation, faithfulness, hallucination]
sources: [Deep Technical Review - Open-Source OCR - Visual Document Understanding Models - Missed Details and Pipeline Implications.md, ocr-arxiv-github-technical-review.md, Open-Source OCR per Visual Document Understanding Models for Self-Hosted AKS Pipelines - A Comparative Investigation (August 2026).md]
created: 2026-08-27
updated: 2026-08-27
---

# CHAOS-Bench

A character-level **hallucination / faithfulness** benchmark: it measures whether a model reproduces what is actually on the page when visual evidence conflicts with language priors. Introduced alongside [[HunyuanOCR]].

## Page-average recall

| Model | Score |
|---|---|
| [[HunyuanOCR]]-1.5 | **14.15** |
| [[DeepSeek-OCR]] 2 | 6.33 |
| [[MinerU]]2.5-Pro | 6.33 |
| [[PaddleOCR-VL]]-1.6 | 5.95 |
| [[GLM-OCR]] | 5.75 |
| [[dots.ocr]] | 3.02 |

## Why this is the metric that matters for high-stakes fields

Every other benchmark rewards *getting the right answer*. CHAOS-Bench rewards *not inventing one*. A model that fluently "corrects" a visually clear but semantically odd string has already destroyed the evidence — no downstream validator can recover it. See [[Linguistic Crutch and Faithfulness]].

Practical consequence: aim the value-in-OCR grounding validator hardest at the fluent-corrector lineage (DeepSeek-derived models), and consider adding CHAOS-Bench-style character-perturbation tests to your golden-set calibration so the confidence ensemble is tuned against faithfulness failures rather than only clean-text accuracy. See [[Confidence Engineering]].

## Related

[[HunyuanOCR]] · [[Linguistic Crutch and Faithfulness]] · [[Confidence Engineering]] · [[PP-OCRv6]]
