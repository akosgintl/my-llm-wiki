---
aliases: ["arXiv and GitHub Technical Review of Missed Details"]
tags: [ocr, primary-sources, architecture, source]
sources: [ocr-arxiv-github-technical-review.md]
created: 2026-08-27
updated: 2026-08-27
---

# arXiv and GitHub Technical Review of Missed Details

**Source:** ocr-arxiv-github-technical-review.md
**Date ingested:** 2026-08-27
**Type:** primary-source review
**Position in the corpus:** seventh document (2026-08-24 23:26) — a per-model pass over the papers and repos, hunting for what the study under-specified.

## Summary

Eleven per-model sections, each structured as links → what the study covers → missed details → implications for the pipeline, with contradictions flagged. Ends with a consolidated link index and a ranked top-ten of missed details.

## Key Claims (the ranked top ten)

1. ✅ **Hungarian is verified in [[PaddleOCR-VL]]'s language list** — upgrading it from "probable" to a confirmed no-fine-tune arm.
2. ⚠️ **PaddleOCR-VL fine-tuning is officially unsupported** — contradicts the study, removes it as a fine-tune target, and strengthens the Qwen/DeepSeek arms.
3. **[[Unsloth]] requires their modified `unsloth/DeepSeek-OCR` checkpoint** — training from the stock repo fails on current transformers.
4. **[[Baidu Unlimited-OCR]]'s 32K budget covers prompt + reference + output together** — the chunking heuristic must subtract vision and prompt tokens; n=128 window and queue-eviction mechanics must match per-request `window_size`.
5. **Qwen3-VL's visual-token pixel budget (256–1280 tokens, 32× compression, ×32 rounding) is the real cost knob** for region crops — and the Hungarian answer sits in **Figure 2** of the paper, a five-minute check.
6. **[[GLM-OCR]]'s GRPO reward menu** (NED/CDM/TEDS/F1 + repetition and malformed-structure penalties) is a copyable objective design; MTP-head survival under LoRA merge is a QA item.
7. **[[DeepSeek-OCR]]'s prompt grammar is byte-sensitive** and its mode/token budgets are fixed per resolution — prompt constants and mode-matched raster sizes belong in CI.
8. **Qwen3.5's Gated DeltaNet hybrid attention invalidates standard KV-cache assumptions** — re-validate MIG memory profiles and pin vLLM versions.
9. **[[MinerU]]2.5-Pro's +5-point gain is data-engineering only** (identical architecture) — the strongest evidence that a data plan beats architecture chasing.
10. **[[HunyuanOCR]]'s faithfulness (CHAOS 14.15) versus DeepSeek's linguistic crutch are opposite failure poles** — pick per field criticality.

## Other notable findings

- [[dots.ocr]]'s **layout-only prompt mode** is a free second-opinion signal for the confidence stack; its repetition triggers are exactly TOC leader dots and form blanks.
- [[MinerU]]'s two stages live inside one 1.2B model, so it needs **no layout-tier placement decision** at all; it preserves headers/footers/page-numbers, making it the archival arm.
- **[[olmOCR]]'s RLVR turns CI fixtures into training rewards** — a third reward paradigm.
- **[[PP-DocLayout-V3]] outputs reading order**, not just boxes — consume it rather than re-sorting by y-coordinate.

## Entities Mentioned

[[GLM-OCR]] · [[PaddleOCR-VL]] · [[DeepSeek-OCR]] · [[Baidu Unlimited-OCR]] · [[dots.ocr]] · [[MinerU]] · [[HunyuanOCR]] · [[olmOCR]] · [[LightOnOCR]] · [[Qwen Model Family]] · [[PP-DocLayout-V3]] · [[OmniDocBench]] · [[EXTRACTCONF]] · [[Unsloth]] · [[glmocr SDK]]

## Concepts Covered

[[R-SWA Reference Sliding Window Attention]] · [[Optical Compression]] · [[Multi-Token Prediction and Speculative Decoding]] · [[Linguistic Crutch and Faithfulness]] · [[Repetition Loops in VLM OCR]] · [[Confidence Engineering]] · [[LoRA Fine-Tuning for OCR]] · [[The OCR-to-Text Boundary Limit]]

## Caveats stated by the source

Compiled from abstracts, HTML excerpts, docs, model cards and repo READMEs — **not cover-to-cover reads**. Items tagged "verify at link" (GLM prompt strings in source, Qwen Figure 2, LightOn training distribution, HunyuanOCR repo path, olmOCR-2 arXiv ID, PP-DocLayoutV3 export formats) are precisely the ones to check by hand. Repos have changed monthly all year.
