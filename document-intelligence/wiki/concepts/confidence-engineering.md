---
aliases: ["Confidence Engineering"]
tags: [confidence, kie, reliability, architecture, hungarian]
sources: [ocr-language-models-deployment-critical-pass-2.md, ocr-vdu-complete-study.md, ocr-arxiv-github-technical-review.md, Deep Technical Review - Open-Source OCR - Visual Document Understanding Models - Missed Details and Pipeline Implications.md]
created: 2026-08-27
updated: 2026-08-27
---

# Confidence Engineering

**Confidence is a pipeline property, not a model feature.**

The uncomfortable baseline: **none of the candidate VLMs natively emit calibrated confidence.** That is a genuine regression against classic OCR — [[Tesseract]], [[PP-OCRv6]] and Azure Document Intelligence all give per-word confidence, and Mistral OCR 4 exposes it via API. Autoregressive decoders emit tokens, not certainty.

Conveniently, engineering it as a pipeline property also makes it **model-swap-proof**, like everything else in [[Pipeline as Platform, Model as Config]].

## The signal stack, cheapest first

**1. Layout detection scores.** [[PP-DocLayout-V3]] boxes already carry confidence. Free. First flag.

**2. vLLM token logprobs** per region or field — aggregate min, mean and P10. Running open weights on [[vLLM]] means you get the *better* estimation family for free: token-probability confidence calibrates markedly better than a model's verbalized self-report (**ECE 27.3 vs 42.7**). ConfBench (arXiv 2608.01792) benchmarks exactly this split for document KIE.

**3. OCR-grounding checks — the highest value per [[EXTRACTCONF]].** A binary **value-in-OCR match** (does the extracted value appear anywhere in the OCR text?) catches hallucinated fields with no grounding at all. Their full system fuses ~40 features through CatBoost into a calibrated score; **you need about 3 of the 40** to get most of it: value-in-OCR + logprob-min + a validator.

**4. Deterministic validators.** For Hungarian business documents: **adóazonosító/adószám checksums, IBAN check digits, date-format and VAT-rate sanity**. Regex-cheap and *perfectly* calibrated for exactly the fields that matter most.

**5. Cross-model agreement.** The [[PP-OCRv6]] hallucination cross-check doubles as a confidence signal; [[dots.ocr]]'s layout-only mode is a free second-opinion input for spatial-divergence features.

**6. Calibration on the golden set.** Temperature scaling or isotonic regression, tracking ECE, so that a threshold **τ becomes the human-review routing knob** — which is what closes the human-in-the-loop question. See [[Golden Set and Eval Harness]].

## The production correction

Extend, a commercial IDP vendor, has been **phasing out raw logprobs-confidence** on newer processors in favor of OCR-grounding plus a review agent. This corroborates the research: **logprobs alone do not survive production; grounding, validators and calibration do.** Do not build the stack logprob-first.

## Risk weighting

Two model-specific priors belong in the scoring:

- **Weight low-visual-token outputs as higher hallucination risk** — fewer vision tokens means more language-prior reliance. See [[Optical Compression]].
- **Aim the value-in-OCR validator hardest at the fluent-corrector lineage** (DeepSeek-derived models). See [[Linguistic Crutch and Faithfulness]].

Add CHAOS-Bench-style character-perturbation tests to the calibration set so the ensemble is tuned against faithfulness failures, not just clean-text accuracy.

## Watch item

**MinerU-Diffusion** decodes by iteratively unmasking positions on per-position confidence — a diffusion OCR decoder that emits confidence *natively*. Incompatible with standard vLLM (custom `nano_dvlm` engine). A quarterly re-check, not a plan. See [[MinerU]].

## Related

[[EXTRACTCONF]] · [[Linguistic Crutch and Faithfulness]] · [[Golden Set and Eval Harness]] · [[Hungarian OCR and the Double Acute Test]] · [[Hallucination Detection in RAG]]
