---
aliases: ["OmniDocBench"]
tags: [benchmarks, evaluation, document-parsing]
sources: [Open-Source OCR per Visual Document Understanding Models for Self-Hosted AKS Pipelines - A Comparative Investigation (August 2026).md, ocr-vdu-complete-study.md, ocr-arxiv-github-technical-review.md, Deep Technical Review - Open-Source OCR - Visual Document Understanding Models - Missed Details and Pipeline Implications.md, Local PDF-Markdown and Document Parsing - State of the Art (August 2026).md]
created: 2026-08-27
updated: 2026-08-27
---

# OmniDocBench

The dominant page-level document-parsing benchmark (arXiv 2412.07626; CVPR 2025 origin, [[OpenDataLab]] / Shanghai AI Lab). Measures edit distance, TEDS and CDM on English + Chinese documents. Now at v1.5/v1.6, which corrected element-matching bias via Multi-Granularity Adaptive Matching and added Base/Hard/Full tiers.

## The vendor leaderboard (Aug 2026)

| Model | v1.5 | v1.6 |
|---|---|---|
| [[PaddleOCR-VL]]-1.6 | — | **96.33** |
| [[MinerU]]2.5-Pro | — | 95.69 |
| [[GLM-OCR]] | 94.62 | 95.15–95.22 |
| [[HunyuanOCR]]-1.5 | — | 94.74 |
| PaddleOCR-VL-1.5 | 94.5 | 94.87–94.93 |
| [[Baidu Unlimited-OCR]] | 93.23 | 93.92 |
| [[DeepSeek-OCR]] 2 | 91.09 | — |
| MinerU2.5 | 90.67 | 92.98 |
| [[dots.ocr]] | ~88.4 | — |
| DeepSeek-OCR v1 | 87.01 | — |

**Read from the official `opendatalab/OmniDocBench` repository on 2026-08-27** (v1.6, updated 2026-04-10; end-to-end `v1.6_full` table): PaddleOCR-VL-1.6 **96.34**, MinerU2.5-Pro **95.75**, GLM-OCR **95.22**, PaddleOCR-VL-1.5 94.93, PaddleOCR-VL 94.18, dots.ocr **90.77**, DeepSeek-OCR 2 **90.25**. Overall = ((1 − text edit distance) × 100 + Table TEDS + Formula CDM) / 3.

**The distinction that matters more than any single score:** [[Baidu Unlimited-OCR]] and [[OvisOCR2]] are **not on the official leaderboard**. Their 93.92 and 96.58 are **self-reported in their own papers**. A self-reported number and a leaderboard entry are not the same kind of evidence, and this wiki previously listed them in one column as if they were. When a self-report outranks the leaderboard leader, that gap is the claim under test — not a result.

## Why you should not trust it

- **It is saturated.** The top open models cluster within ~2 points at 94–96. LlamaIndex has publicly called it saturated.
- **It is vendor-reported.** Models typically self-evaluate while competitors' numbers are pulled from the OmniDocBench repo.
- **Its composition skews academic and English/Chinese print**, which is exactly the bias that inverts on multi-script evaluation. Read §data-composition of the paper to see it.
- **Its edit-distance metric is formatting-sensitive**, which is part of why a structurally sound framework like [[Docling]] scores badly on it.

Treat leaderboard deltas of 1–2 points as noise. See [[Benchmark Saturation]] — and note that the only leaderboard that counts is your own [[Golden Set and Eval Harness]].

## Related

[[olmOCR-Bench]] · [[OCR Arena]] · [[Multi-Script OCR Benchmarks]] · [[CHAOS-Bench]] · [[Benchmark Saturation]]
