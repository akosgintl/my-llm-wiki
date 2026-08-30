---
aliases: ["Linguistic Crutch and Faithfulness"]
tags: [failure-modes, hallucination, faithfulness, ocr]
sources: [ocr-arxiv-github-technical-review.md, Deep Technical Review - Open-Source OCR - Visual Document Understanding Models - Missed Details and Pipeline Implications.md, ocr-vdu-complete-study.md, Open-Source OCR per Visual Document Understanding Models for Self-Hosted AKS Pipelines - A Comparative Investigation (August 2026).md]
created: 2026-08-27
updated: 2026-08-27
---

# Linguistic Crutch and Faithfulness

VLM OCR models have a language model inside them, and it does not stop working when the pixels are clear. The result is a failure mode classic OCR never had: **the model silently "corrects" visually unambiguous but semantically odd text.**

## The evidence

The critique paper *"Visual Merit or Linguistic Crutch?"* (arXiv 2601.03714) evaluated [[DeepSeek-OCR]] against 13 baselines and found:

- Under sentence- and word-level **semantic corruption**, accuracy plummets from **~90% to ~20%**.
- **Lower visual-token counts correlate with more reliance on language priors and higher hallucination.**
- **Traditional pipeline OCR methods are significantly more robust than end-to-end methods** on this axis.

The second point is the operationally important one: it means the resolution mode you choose is also a hallucination-risk setting. See [[Optical Compression]].

## The two poles

[[CHAOS-Bench]] measures exactly this — character-level faithfulness under visual-versus-prior conflict:

| Model | Score | Pole |
|---|---|---|
| [[HunyuanOCR]]-1.5 | **14.15** | faithful — reproduces the source *including its errors* |
| [[DeepSeek-OCR]] 2 | 6.33 | fluent corrector |
| [[MinerU]]2.5-Pro | 6.33 | |
| [[PaddleOCR-VL]]-1.6 | 5.95 | |
| [[GLM-OCR]] | 5.75 | |
| [[dots.ocr]] | 3.02 | most fluent |

**Pick per field criticality.** For high-stakes fields, a faithful model plus a downstream text-LLM correction beats a fluent model that has already corrupted the evidence — because once the model has written a plausible wrong value, no validator can recover the original.

## Why this is worse than an ordinary error

A random OCR error produces garbage, which is detectable. A linguistic-crutch error produces a **plausible, well-formed, wrong value** — a real Hungarian word, a valid-looking amount, a well-formed name. It passes every format check.

For Hungarian specifically, low-visual-evidence outputs will hallucinate *plausible Hungarian words*, and the double-acute characters are exactly where prior-driven substitution lands. See [[Hungarian OCR and the Double Acute Test]].

## Defenses

1. **Value-in-OCR grounding** — the single highest-value confidence feature. Aim it hardest at the fluent-corrector lineage. See [[Confidence Engineering]].
2. **Deterministic validators** on numeric and ID fields — checksums cannot be talked out of a wrong value.
3. **[[PP-OCRv6]] as a cross-check** — a 34.5M CTC recognizer with no language model cannot invent a word, so disagreement with it is signal.
4. **Weight low-visual-token outputs as higher hallucination risk** in the confidence model, and add CHAOS-Bench-style character-perturbation tests to golden-set calibration.

## Related

[[CHAOS-Bench]] · [[Confidence Engineering]] · [[HunyuanOCR]] · [[PP-OCRv6]] · [[Repetition Loops in VLM OCR]]
