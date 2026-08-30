---
aliases: ["HunyuanOCR"]
tags: [ocr, vlm, multilingual, faithfulness, model]
sources: [Open-Source OCR per Visual Document Understanding Models for Self-Hosted AKS Pipelines - A Comparative Investigation (August 2026).md, ocr-language-models-deployment-critical-pass-2.md, ocr-vdu-complete-study.md, ocr-arxiv-github-technical-review.md, Deep Technical Review - Open-Source OCR - Visual Document Understanding Models - Missed Details and Pipeline Implications.md]
created: 2026-08-27
updated: 2026-08-27
---

# HunyuanOCR

~1B end-to-end document model from [[Tencent]] (v1.5: arXiv 2607.04884; v1.0: arXiv 2511.19575). Adds DFlash speculative decoding and an Agentic Data Flow over v1.0, backbone unchanged.

## The faithfulness outlier

On **[[CHAOS-Bench]]**, which measures character-level hallucination under visual-versus-prior conflict, HunyuanOCR-1.5 scores **14.15** page-average recall against:

| Model | CHAOS-Bench |
|---|---|
| **HunyuanOCR-1.5** | **14.15** |
| [[DeepSeek-OCR]] 2 | 6.33 |
| [[MinerU]]2.5-Pro | 6.33 |
| [[PaddleOCR-VL]]-1.6 | 5.95 |
| [[GLM-OCR]] | 5.75 |
| [[dots.ocr]] | 3.02 |

It reproduces the source *including its errors and odd spellings* rather than silently correcting them — the exact opposite failure pole from DeepSeek's [[Linguistic Crutch and Faithfulness]] problem. For high-stakes fields, a faithful model plus a downstream text-LLM correction beats a fluent model that has already corrupted the evidence.

## Other strengths

- OmniDocBench v1.6 overall **94.74**.
- Top open model on the **MORE** multilingual benchmark (92.42), which spans 149 low-resource languages — though Hungarian membership is not individually confirmed. See [[Multi-Script OCR Benchmarks]].
- Built-in **text-image translation** (source-language page → target-language text), a route for Hungarian→English delivery without a second hop.
- Coordinate/grounding outputs; unified document parsing, text spotting, information extraction, and multi-image document QA. llama.cpp deployment for PCs.

## Cautions

- **License is NOASSERTION** — needs counsel review before commercial use. See [[Permissive Licensing Constraints]].
- The repo notes a **[[vLLM]]-versus-transformers accuracy gap under repair**. Pin and A/B the engines before trusting any vLLM-measured bake-off score for this model.

## Verdict

The strongest candidate for the "evidence-preserving" arm of a [[Confidence Engineering]] architecture, and a serious Hungarian contender. Benchmark it head-to-head against the primary on the [[Golden Set and Eval Harness]] — but verify engine parity first.

## Related

[[Linguistic Crutch and Faithfulness]] · [[CHAOS-Bench]] · [[PP-OCRv6]] · [[Tencent]]
