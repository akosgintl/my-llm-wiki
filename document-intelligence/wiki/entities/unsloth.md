---
aliases: ["Unsloth"]
tags: [fine-tuning, lora, tool]
sources: [ocr-language-models-deployment-critical-pass-2.md, ocr-vdu-complete-study.md, ocr-arxiv-github-technical-review.md, Deep Technical Review - Open-Source OCR - Visual Document Understanding Models - Missed Details and Pipeline Implications.md]
created: 2026-08-27
updated: 2026-08-27
---

# Unsloth

The fine-tuning framework with the best OCR tooling maturity, and therefore a decision-driver in its own right — tooling maturity settles more here than architecture does.

## Maturity ranking across candidates

| Path | Rating | Notes |
|---|---|---|
| **[[DeepSeek-OCR]] via Unsloth** | ★★★ | Official support, free Colab notebooks (+ an eval variant), 1.4× faster and −40% VRAM |
| **[[Qwen Model Family]] (3.5) via Unsloth / [[LLaMA-Factory]]** | ★★★ | Day-1 framework support, first-class transformers integration, no custom-code surgery |
| [[GLM-OCR]] via LLaMA-Factory | ★★ | Official in-repo tutorial (Feb 2026) — but you would be teaching a language it barely has, from the worst starting point |
| [[PaddleOCR-VL]] | ★ | Officially **unsupported**; the docs FAQ says fine-tuning is planned, not released |
| [[Baidu Unlimited-OCR]] | — | No tooling, but it *is* the recipe template (continue-trained from DeepSeek-OCR, encoder frozen, decoder only) |

## The critical gotcha

The **stock `deepseek-ai/DeepSeek-OCR` checkpoint does not run or train on current transformers**. Unsloth ships a modified upload (`unsloth/DeepSeek-OCR`, `unsloth/DeepSeek-OCR-2`, incorporating Stranger Vision HF changes). Fine-tune from *that*, not from the vendor repo.

## The Persian existence proof

Unsloth's demo: a 200K-sample dataset produced an **88.26% absolute CER improvement (149.07% → 60.81%)**, with **most of the gain arriving after 60 steps at batch 8 (~480 samples seen)**. Persian is a *new script*; Hungarian is Latin plus new diacritics, i.e. strictly easier.

**Honesty note:** these are the vendor's own numbers on a different script. Treat "60 steps fixed it" as proof the mechanism works, not as a promise about any specific corpus. See [[LoRA Fine-Tuning for OCR]].

## Related

[[LLaMA-Factory]] · [[DeepSeek-OCR]] · [[LoRA Fine-Tuning for OCR]] · [[Golden Set and Eval Harness]]
