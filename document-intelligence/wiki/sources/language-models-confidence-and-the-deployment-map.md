---
aliases: ["Language Models Confidence and the Deployment Map"]
tags: [hungarian, fine-tuning, confidence, infrastructure, source]
sources: [ocr-language-models-deployment-critical-pass-2.md]
created: 2026-08-27
updated: 2026-08-27
---

# Language Models Confidence and the Deployment Map

**Source:** ocr-language-models-deployment-critical-pass-2.md
**Date ingested:** 2026-08-27
**Type:** critical analysis
**Position in the corpus:** fifth document (2026-08-24 23:05) — **the pivot point of the whole project.** This is where the Hungarian constraint arrives and re-ranks everything.

## Summary

A second critical pass covering the open design decisions, GPU-cloud alternatives and EU sovereignty, the Hungarian pivot, fine-tuning strategy, layout-stage economics, the glmocr-SDK model swap, the Qwen family map, downstream text-LLM roles, and confidence engineering.

## Key Claims

- **The critique that matters most:** *"We designed the supply side before interrogating the demand side — and the demand side promptly invalidated a core choice."* One fact — Hungarian is 80% of the corpus — demoted [[GLM-OCR]] from default workhorse to niche tool in a single sentence.
- **Rule 0: the golden set is the most valuable artifact in the project.** 50–100 labeled Hungarian pages plus an eval harness (CER, word accuracy, diacritic-specific error rate, TableTEDS) costs one engineer-week and converts every future decision from vibes to measurement. **Nothing else is worth doing before it.**
- **Rule 0.5: keep the pipeline the fixed point, make the model the replaceable part.** This inversion is what makes the ~2-month model treadmill irrelevant instead of exhausting.
- **The ő/ű test.** Hungarian's double acutes are unique to the language and are the canonical failure; plain CER barely registers a substitution that corrupts every downstream field. The diacritic-specific error rate is the tiebreaker.
- **"EU region of a US cloud" ≠ sovereign** — CLOUD Act exposure follows *incorporation*, not server location. Recommended posture: AKS control plane + Scaleway/OVHcloud sovereign burst + RunPod/Modal for non-sensitive work.
- **Fine-tune recipe:** freeze the vision encoder, LoRA the decoder — Hungarian is a language problem, not a vision problem. Born-digital PDFs are free labels; synthetic rendering scales the tail; keep 10–30% replay data to avoid catastrophic forgetting.
- **Format-native fine-tuning dissolves two problems at once** — one run buys the language *and* the output contract.
- **The killer discovery:** the glmocr SDK maps dotted config paths to `GLMOCR_`-prefixed env vars, so repointing it at any vLLM service is a pure ConfigMap change.
- **Confidence is engineered, not emitted.** None of the candidate VLMs emit calibrated confidence — a genuine regression versus classic OCR.
- **The measured ceiling of OCR+text-LLM:** CharXiv 0.0 versus 81.8 end-to-end. Anything visual dies at the boundary.

## Entities Mentioned

- [[GLM-OCR]] — disqualified as default; the SDK survives
- [[glmocr SDK]] — the env-var override discovery and the three swap options
- [[Qwen Model Family]] — Qwen3.5-4B becomes the front-runner
- [[PaddleOCR-VL]], [[dots.ocr]], [[HunyuanOCR]], [[LightOnOCR]], [[DeepSeek-OCR]] — the Hungarian status board
- [[Unsloth]], [[LLaMA-Factory]] — fine-tuning tooling ranking
- [[PP-DocLayout-V3]] — the layout tier economics
- [[EXTRACTCONF]] — the confidence feature menu
- [[neural-maze production-ocr-course]] — decoded, with a copy-with-eyes-open list

## Concepts Covered

- [[Golden Set and Eval Harness]] (Rule 0) · [[Pipeline as Platform, Model as Config]] (Rule 0.5)
- [[Hungarian OCR and the Double Acute Test]] — the pivot
- [[LoRA Fine-Tuning for OCR]] · [[Born-Digital Self-Labeling]]
- [[EU Sovereign GPU Hosting]] · [[Layout Stage Economics]]
- [[Confidence Engineering]] · [[The OCR-to-Text Boundary Limit]]

## Caveats stated by the source

EU pricing and certification claims are snapshots. Hungarian membership in PaddleOCR-VL's 109-language list was still "probable, unverified" here — **later resolved to verified**. Unsloth's Persian numbers are a vendor demo on a different script. Layout-stage latencies are RT-DETR-class estimates, not published benchmarks.
