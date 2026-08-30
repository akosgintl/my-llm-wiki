---
aliases: ["Granite-Docling"]
tags: [ocr, vlm, document-parsing, model]
sources: [Local PDF-Markdown and Document Parsing - State of the Art (August 2026).md, Open-Source OCR per Visual Document Understanding Models for Self-Hosted AKS Pipelines - A Comparative Investigation (August 2026).md, tiered-pdf-pipeline-architecture.md]
created: 2026-08-27
updated: 2026-08-27
---

# Granite-Docling

[[IBM Research]]'s 258M document VLM, released **2025-09-17**, Apache-2.0. The smallest serious parser in the field and the VLM path inside [[Docling]] (`--pipeline vlm`).

## Lineage

Per IBM's announcement, a "product-ready evolution of the experimental SmolDocling-256M-preview model released in March 2025": the SmolLM-2 backbone was replaced with a **Granite 3-based** architecture (Granite 165M LLM), and the SigLIP encoder swapped for **SigLIP2** (siglip2-base-patch16-512), on an Idefics3-derived architecture. Outputs IBM's **DocTags** markup.

Improvements over SmolDocling: enhanced equation recognition, flexible inference modes (full-page or bbox-guided region), and improved stability with fewer infinite token-repetition loops. Backends: transformers, [[vLLM]], ONNX, MLX.

## Reported metrics (HF model card)

| Metric | Score |
|---|---|
| Layout F1 | 0.86 |
| Full-page OCR F1 / edit distance | 0.84 / 0.45 |
| Code F1 / edit | 0.988 / 0.013 |
| Equation F1 / edit | 0.968 / 0.073 |
| Table TEDS-structure / with content | 0.97 / 0.96 (FinTabNet 150dpi) |
| OCRBench | 500 |

The vision model was trained on 81,000 manually labeled pages (patents, manuals, 10-K filings).

## Limits

Multilingual with English primary; Chinese, Japanese and Arabic are experimental. **Hungarian is not a claimed language.** At 258M its absolute accuracy on hard tables and formulas trails the 1–2B specialists, so it narrows but does not close [[Docling]]'s OmniDocBench gap. It appears in [[Tiered Page Routing]] designs only as an Apache-2.0 swap candidate for the VLM tier, behind [[PaddleOCR-VL]].

## Related

[[Docling]] · [[IBM Research]] · [[PaddleOCR-VL]]
