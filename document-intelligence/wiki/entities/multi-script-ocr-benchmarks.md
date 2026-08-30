---
aliases: ["Multi-Script OCR Benchmarks"]
tags: [benchmarks, evaluation, multilingual]
sources: [Open-Source OCR per Visual Document Understanding Models for Self-Hosted AKS Pipelines - A Comparative Investigation (August 2026).md, ocr-language-models-deployment-critical-pass-2.md, ocr-vdu-complete-study.md, Deep Technical Review - Open-Source OCR - Visual Document Understanding Models - Missed Details and Pipeline Implications.md]
created: 2026-08-27
updated: 2026-08-27
---

# Multi-Script OCR Benchmarks

The independent evaluations that **reorder the field** against [[OmniDocBench]]. For any non-English, non-CJK corpus these matter more than the vendor leaderboard.

## socOCRbench

Noah Dasanaike (Harvard). 280 full-page multi-region/multi-script images; overall = mean of NES, chrF and TEDS; updated through June 2026.

| System | Score |
|---|---|
| Gemini 3.1 Pro (low) — best proprietary | 0.6357 |
| [[dots.ocr]] 1.5 | ~0.478 |
| [[PaddleOCR-VL]]-1.6 | ~0.394 |
| [[GLM-OCR]] | ~0.368 |
| [[DeepSeek-OCR]] 2 | ~0.176 |
| [[MinerU]]2.5-Pro | ~0.165 |
| Tesseract v5 | ~0.098 |
| DeepSeek-OCR v1 | ~0.086 |

DeepSeek-OCR v1 scoring **below Tesseract** is the headline: models overfit to clean print collapse on real-world multi-script pages.

## MDPBench

arXiv 2603.28130 — **3,400 document images across 17 languages**, real-world rather than clean-print. Added 2026-08-27; the roster is not enumerated in the abstract and **Hungarian membership is unconfirmed**.

Its headline finding is the one this page exists to make: open-source models drop **17.8% on photographed documents and 14.0% on non-Latin scripts**, while closed-source systems hold up. [[Qwen Model Family]] Qwen3-VL-8B reports MDPBench overall 68.3 (digital 78.4, photographed 65.0) on its own card — the same digital-versus-photographed cliff, self-reported.

Worth checking the language roster before the probe: if Hungarian is among the 17, this becomes the only real-world multilingual benchmark with direct Hungarian evidence.

## GlotOCR Bench

arXiv 2604.12978, mid-resource scripts. Gemini 3.1 Flash-Lite leads (CER 27.7, Acc@5 61.9%) with **dots.ocr second**. GLM-OCR and DeepSeek-OCR-2 sit **over 40 points below** the leader — a categorical gap, not a marginal one.

## MORE

Multilingual parsing benchmark spanning **149 low-resource languages**.

| Model | Score |
|---|---|
| [[HunyuanOCR]] | **92.42** (top open model) |
| PaddleOCR-VL | 87.96 |
| dots.ocr | 84.31 |
| MinerU | 48.85 (last by a wide margin) |

Hungarian membership in MORE's 149 is not individually confirmed.

## What to take from them

1. The independent ranking is roughly **dots.ocr > PaddleOCR-VL ≈ HunyuanOCR ≫ GLM-OCR > DeepSeek/MinerU** — nearly the inverse of the vendor board.
2. Proprietary systems still lead independent multi-script evaluation by a wide margin, which is the argument for keeping a managed API as a hard-case fallback if compliance permits.
3. None of these break out Hungarian. They narrow the candidate list; they do not settle it. See [[Hungarian OCR and the Double Acute Test]].

## Related

[[OmniDocBench]] · [[OCR Arena]] · [[Benchmark Saturation]] · [[Golden Set and Eval Harness]]
