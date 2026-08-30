---
aliases: ["Deep Technical Review of Missed Details and Pipeline Implications"]
tags: [ocr, primary-sources, hungarian, licensing, source]
sources: [Deep Technical Review - Open-Source OCR - Visual Document Understanding Models - Missed Details and Pipeline Implications.md]
created: 2026-08-27
updated: 2026-08-27
---

# Deep Technical Review of Missed Details and Pipeline Implications

**Source:** Deep Technical Review - Open-Source OCR - Visual Document Understanding Models - Missed Details and Pipeline Implications.md
**Date ingested:** 2026-08-27
**Type:** primary-source review
**Position in the corpus:** eighth document (2026-08-25 09:16) — a second, deeper pass over the same papers as source 7, with more exact numbers and an ecosystem-repo verdict list.

## Summary

Eleven per-model sections plus an "adjacent / implementation-relevant" section, a judged list of ecosystem repos, a consolidated link index, a ranked top ten, an explicit contradictions list, and dated recommendations.

## Key Claims

- **The single most consequential finding:** GLM-OCR officially claims only **8 languages** (zh, en, fr, es, ru, de, ja, ko) — **Hungarian is not among them**, and the "100+ languages" claim is unverified marketing absent from the paper, README and model card. LoRA fine-tuning becomes mandatory rather than optional.
- **[[PaddleOCR-VL]] explicitly lists Hungarian** in its 109→111-language Latin set — **but does not support fine-tuning.** These two facts together make it the *ceiling-fixed* arm: a benchmark oracle and second-opinion signal rather than the trainable primary.
- **GLM-OCR's MTP survival under LoRA merge is undocumented** in repo, paper or issues — an open risk to the ~50% speculative-decoding throughput gain that must be validated empirically.
- **DeepSeek-OCR's AGPL-3.0 PyMuPDF dependency legally contaminates its MIT claim** (Issue #223) — a genuine commercial blocker.
- **The DeepSeek critique paper directly validates the confidence stack**: accuracy plummets ~90%→~20% under semantic corruption, and fewer visual tokens means more hallucination.
- **[[HunyuanOCR]]'s CHAOS-Bench 14.15 versus 5–6 for peers** is the highest-impact anti-hallucination signal available.
- **[[PP-DocLayout-V3]] has validated community ONNX exports**, enabling a Paddle-free CPU layout tier — **with a critical gotcha**: the ONNX model expects `mean=[0,0,0], std=[1,1,1]`, **not ImageNet normalization**; getting it wrong drops detections.
- **Exact GLM-OCR prompt strings** are pinned down: `Text Recognition:` / `Formula Recognition:` / `Table Recognition:`, with `<image>` placed **before** the text prompt. KIE has no canonical string.
- **Qwen3-VL's language marketing is inconsistent** ("32 up from 10" vs "up from 19"); the paper's 39 OCR languages with >70% accuracy on 32 is authoritative. The Qwen3.5-Omni Fleurs Hungarian listing is **speech, not OCR**.

## Ecosystem repos, judged

**Highest value:** [[neural-maze production-ocr-course]] — near-exact topological match, but free course material, not battle-tested. **Use:** vLLM Recipes (authoritative MTP flags), [[Unsloth]], [[LLaMA-Factory]]'s GLM-OCR recipe (validate MTP post-merge), OmniDocBench/olmOCR-bench harnesses, the PP-DocLayoutV3 ONNX exports, `zhaohb/paddleocr_vl_ov`. **Skip:** generic awesome-OCR lists, one-off Gradio demo Spaces, aggregator DeepWiki pages.

## Entities Mentioned

[[GLM-OCR]] · [[PaddleOCR-VL]] · [[DeepSeek-OCR]] · [[Baidu Unlimited-OCR]] · [[dots.ocr]] · [[MinerU]] · [[HunyuanOCR]] · [[olmOCR]] · [[LightOnOCR]] · [[Qwen Model Family]] · [[PP-DocLayout-V3]] · [[CHAOS-Bench]] · [[OmniDocBench]] · [[EXTRACTCONF]] · [[neural-maze production-ocr-course]] · [[Unsloth]] · [[LLaMA-Factory]]

## Concepts Covered

[[Hungarian OCR and the Double Acute Test]] · [[Multi-Token Prediction and Speculative Decoding]] · [[Permissive Licensing Constraints]] · [[Linguistic Crutch and Faithfulness]] · [[Optical Compression]] · [[Confidence Engineering]] · [[Layout Stage Economics]] · [[R-SWA Reference Sliding Window Attention]]

## Explicit contradictions it raises

Against the study: PaddleOCR-VL is **not** trainable/swappable (docs say fine-tuning unsupported); Hungarian membership is confirmed **only** for PaddleOCR-VL and likely absent for GLM-OCR, DeepSeek-OCR and LightOnOCR-2; Qwen3-VL's language count is misreported in marketing; GLM-OCR's "100+ languages" is unverified.

## Caveats stated by the source

2026 arXiv IDs are preprints; several benchmark numbers are self-reported and should be reproduced on the Hungarian golden set. GLM-OCR's MTP-under-merge behavior and the CogViT connector downsampling ratio are **genuinely undocumented, not merely unfound**.
