---
aliases: ["LightOnOCR"]
tags: [ocr, vlm, multilingual, model]
sources: [Open-Source OCR per Visual Document Understanding Models for Self-Hosted AKS Pipelines - A Comparative Investigation (August 2026).md, ocr-language-models-deployment-critical-pass-2.md, ocr-vdu-complete-study.md, ocr-arxiv-github-technical-review.md, Deep Technical Review - Open-Source OCR - Visual Document Understanding Models - Missed Details and Pipeline Implications.md]
created: 2026-08-27
updated: 2026-08-27
---

# LightOnOCR

LightOn's 1B end-to-end parser (arXiv 2601.14251, `lightonai/LightOnOCR-2-1B`), Apache-2.0. Marketed on European-language coverage — but that coverage **does not include Hungarian** (verified below).

## Specs

- Pretraining scaled 17M→43M pages; teacher was Qwen3-VL-235B-A22B-Instruct. Max longest edge raised 1024→**1540 px**.
- Trained with RLVR plus task-arithmetic ("soup") merging.
- **83.2 ± 0.9 on [[olmOCR-Bench]]**; 5.71 pages/s on one H100 (~493k pages/day, under $0.01 per 1k pages). Some sources report ~42.8 pages/s on H100 as a peak figure.
- Variants: OCR-only, `-bbox` (bounding boxes for embedded images), `-bbox-soup`, plus base checkpoints for RLVR.
- 3% blank-page examples were included in training specifically to prevent hallucination loops — see [[Repetition Loops in VLM OCR]].
- Standard autoregressive, vanilla-[[vLLM]] servable, no custom logits processors.

## The Hungarian question

**Resolved 2026-08-27 — Hungarian is NOT declared.** The `lightonai/LightOnOCR-2-1B` model card declares **11 languages**: `en fr de es it nl pt sv da zh ja`. There is **no Central or Eastern European language at all** — no Hungarian, Polish, Czech or Romanian. (The earlier `LightOnOCR-1B-1025` card declares 9: the same list minus `zh ja`.) This matches §6 of the paper, which states multilingual performance outside the trained European set is not fully supported.

**Consequence:** LightOnOCR drops out of the Hungarian shortlist. It remains useful as a **throughput and cost baseline** — and as a fine-tune base, since Hungarian is Latin-script and the decoder already handles Western European diacritics. But it is no longer a zero-work probe. See [[Hungarian OCR and the Double Acute Test]] and [[LoRA Fine-Tuning for OCR]].

## Verdict

**Demoted.** It was the cheapest Hungarian probe only while Hungarian was an open question; the model card closes that question negatively. What survives is a strong **throughput and cost reference point** (5.71 pages/s on one H100, under $0.01 per 1k pages) and a credible Latin-script fine-tune base. Probe it only after the models that actually declare Hungarian.

## Related

[[olmOCR]] · [[dots.ocr]] · [[Golden Set and Eval Harness]]
