---
aliases: ["PP-OCRv6"]
tags: [ocr, recognizer, hallucination, model]
sources: [Open-Source OCR per Visual Document Understanding Models for Self-Hosted AKS Pipelines - A Comparative Investigation (August 2026).md, ocr-language-models-deployment-critical-pass-2.md, ocr-vdu-complete-study.md]
created: 2026-08-27
updated: 2026-08-27
---

# PP-OCRv6

[[Baidu]]'s tiny 34.5M CTC+NRTR text recognizer — the classic, pre-VLM kind of OCR. It appears in a modern VLM architecture for one specific reason.

## The hallucination-defense role

PP-OCRv6 **faithfully reproduces misspellings without linguistic-prior hallucination**. It has no language model to "correct" with, so it cannot invent a plausible word where the pixels say something odd.

That makes it the natural cross-check against the [[Linguistic Crutch and Faithfulness]] failure of VLM parsers. Two uses:

1. **A hallucination cross-check for high-stakes fields** (finance, legal, medical), where a VLM's fluent correction of a name, amount or ID is unrecoverable damage.
2. **A confidence signal** — cross-model agreement between the VLM output and PP-OCRv6 is signal #5 in the [[Confidence Engineering]] stack, and it is nearly free.

It is also the recognizer inside [[MinerU]]'s CPU-capable `pipeline` backend.

## Related

[[Confidence Engineering]] · [[HunyuanOCR]] · [[Linguistic Crutch and Faithfulness]] · [[Tesseract]] · [[Baidu]]
