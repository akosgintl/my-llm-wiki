---
aliases: ["Tesseract"]
tags: [ocr, recognizer, cpu, hungarian, tool]
sources: [tiered-pdf-pipeline-architecture.md, Local PDF-Markdown and Document Parsing - State of the Art (August 2026).md, ocr-language-models-deployment-critical-pass-2.md]
created: 2026-08-27
updated: 2026-08-27
---

# Tesseract

The classic open-source OCR engine. Largely superseded on accuracy — [[DeepSeek-OCR]] v1 scoring *below* Tesseract v5 on socOCRbench is a comment on DeepSeek, not a recommendation of Tesseract — but it holds two live roles.

## Role 1: Tier 2 of a Hungarian-first pipeline

In [[Tiered Page Routing]], Tesseract with the `hun` language pack sits inside the [[Docling]] + TableFormer tier for tables and multi-column pages on CPU. **Set `lang=hun` explicitly — never rely on autodetect.**

Notably, **Tesseract `hun` handles the ő/ű double acute well on clean scans**. It is the VLM output on *degraded* scans where diacritic drift appears. That makes Tesseract a useful reference point rather than a fallback of last resort. See [[Hungarian OCR and the Double Acute Test]].

## Role 2: calibrated per-word confidence

Tesseract emits per-word confidence, as do [[PP-OCRv6]] and Azure Document Intelligence. **None of the candidate VLMs do.** That regression is the whole reason confidence has to be engineered as a pipeline property — see [[Confidence Engineering]].

It is also the OCR engine available inside [[liteparse]].

## Related

[[Tiered Page Routing]] · [[Docling]] · [[PP-OCRv6]] · [[Confidence Engineering]]
