---
aliases: ["EXTRACTCONF"]
tags: [confidence, kie, evaluation, benchmarks]
sources: [ocr-language-models-deployment-critical-pass-2.md, ocr-vdu-complete-study.md, ocr-arxiv-github-technical-review.md]
created: 2026-08-27
updated: 2026-08-27
---

# EXTRACTCONF

arXiv 2606.24420 — the reference work on **calibrated confidence for document key-information extraction**, and the source of the highest-value signal in the [[Confidence Engineering]] stack.

## The system

~40 features fused through CatBoost into a calibrated per-field score, with optional post-hoc recalibration, routing each field to automation or human review at a threshold τ. The feature menu:

- logprob mean / min / P10 / std per call
- entropy mean / max / P90 per call
- **value-in-OCR match** (binary: does the extracted value appear anywhere in the OCR text?)
- neighborhood-text overlap
- spatial centroid divergence
- OCR confidence
- image quality

## The 3-of-40 shortcut

You do not need all forty. Three features capture most of the value:

1. **value-in-OCR match** — catches hallucinated fields with no grounding at all
2. **logprob-min** for the field
3. **a deterministic validator** (checksum, format rule)

The full list is the upgrade path once the shortcut plateaus.

## Companion: ConfBench

arXiv 2608.01792 benchmarks **verbalized vs logprob confidence** for document KIE across three input-modality conditions. Its finding underpins the design: token-probability confidence calibrates markedly better than a model's verbalized self-report (**ECE 27.3 vs 42.7**). Its appendix A.5 workflow is a ready-made template for a confidence evaluation harness.

## The production counterweight

Extend (a commercial IDP vendor) has been **phasing out raw logprobs-confidence** on newer processors in favor of OCR-grounding plus a review agent. Logprobs alone do not survive production; grounding, validators and calibration do.

## Related

[[Confidence Engineering]] · [[CHAOS-Bench]] · [[Golden Set and Eval Harness]]
