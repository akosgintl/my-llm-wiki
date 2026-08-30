---
aliases: ["Golden Set and Eval Harness"]
tags: [evaluation, methodology, hungarian, benchmarks]
sources: [ocr-language-models-deployment-critical-pass-2.md, ocr-vdu-complete-study.md, Deep Technical Review - Open-Source OCR - Visual Document Understanding Models - Missed Details and Pipeline Implications.md, Open-Source OCR per Visual Document Understanding Models for Self-Hosted AKS Pipelines - A Comparative Investigation (August 2026).md]
created: 2026-08-27
updated: 2026-08-27
---

# Golden Set and Eval Harness

**Rule 0: the golden set is the most valuable artifact in the project.**

50–100 labeled pages from the real corpus plus an eval harness converts every subsequent decision — which model, whether to fine-tune, which cloud, which confidence threshold, when a new release matters — from vibes into measurement. Cost: about one engineer-week. **Nothing else is worth doing before it.**

## The four metrics

| Metric | Why |
|---|---|
| CER (character error rate) | overall baseline |
| Word accuracy | closer to downstream field extraction |
| **Diacritic-specific error rate** | the tiebreaker — see below |
| TableTEDS | structure, not just text |

Plus per-page latency when comparing serving configurations.

## Why the third metric decides everything

Plain CER **barely registers** a silent ő→ö substitution — it is a single character in a page of thousands — while that substitution corrupts every downstream field and search index. Counting á/é/í/ó/ú/ö/ü/ő/ű confusions as their own metric is what makes the failure visible. See [[Hungarian OCR and the Double Acute Test]].

## Why it replaces the public leaderboards

[[OmniDocBench]] is saturated, vendor-reported and Latin/CJK-print-biased. [[olmOCR-Bench]] is English-only and its header/footer category can reward omitting content. [[OCR Arena]] and the [[Multi-Script OCR Benchmarks]] reorder the field sharply but break out no Hungarian. **Vendor deltas of 1–2 points mean nothing; your golden set is the only leaderboard that counts.** See [[Benchmark Saturation]].

## What it unblocks

Every open question resolves to "run it against the golden set and read the diacritic column":

- the **probe bake-off** — 30–50 real pages, all candidates via their cheapest inference path, two days of work. It may end the fine-tuning conversation before it starts: if the best off-the-shelf CER is already low-single-digit with clean diacritics, ship it.
- **fine-tune gating** — every checkpoint judged on the held-out set with the diacritic metric, not CER alone. See [[LoRA Fine-Tuning for OCR]].
- **confidence calibration** — temperature scaling or isotonic regression, tracking ECE, so the threshold τ becomes the human-review routing knob. See [[Confidence Engineering]].
- **the version treadmill** — a new checkpoint is one `SERVED_NAME` change plus one golden-set run from a verdict, never an adoption decision made on a release blog. See [[Pipeline as Platform, Model as Config]].

## Build it cheaply

[[Born-Digital Self-Labeling]] produces the labeled pairs at zero annotation cost, and the same pipeline feeds both the golden set and any training set. Build it regardless of the fine-tune decision.

Consider adding **CHAOS-Bench-style character-perturbation tests** to the calibration so the ensemble is tuned against faithfulness failures rather than only clean-text accuracy. See [[CHAOS-Bench]].

## The RAG-side equivalent

The same discipline applies downstream: build a golden query set from real user queries early, measure retrieval metrics separately from generation metrics, and gate every change in CI. See [[RAG Evaluation]].

What the golden set unblocks, and what else is still missing, is catalogued in [[Open Inputs and Corpus Profile]].

## Related

[[Benchmark Saturation]] · [[Hungarian OCR and the Double Acute Test]] · [[LoRA Fine-Tuning for OCR]] · [[Confidence Engineering]] · [[RAG Evaluation]]
