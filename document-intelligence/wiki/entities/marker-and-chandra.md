---
aliases: ["Marker and Chandra"]
tags: [document-parsing, vlm, licensing, tool]
sources: [Local PDF-Markdown and Document Parsing - State of the Art (August 2026).md, tiered-pdf-pipeline-architecture.md]
created: 2026-08-27
updated: 2026-08-27
---

# Marker and Chandra

Datalab's parsing stack. **marker** is a pipeline built around the **Surya** VLM inference server plus small CPU models (pdftext, a 20M layout model). **Chandra 2** (March 2026) is their 4B full-page VLM.

## Numbers

| System | [[olmOCR-Bench]] | Notes |
|---|---|---|
| Chandra 2 | **85.8** | SOTA at its class; runs on ~10–12 GB VRAM |
| Marker 2 | 76.0 overall / 83.5 born-digital | 2.9 pages/s on a B200; fast/no-OCR CPU path ~23.7 pages/s (43.6 on bench) |
| "Datalab Marker" | 83.2 | #2 on the Nanonets leaderboard |

## The licensing blocker

Code is Apache-2.0, but the **weights are a modified AI Pubs Open Rail-M**. **Corrected 2026-08-27:** the `datalab-to/marker` README now reads *"Our model weights use a modified AI Pubs Open Rail-M license (free for research, personal use, and startups under $5M funding/revenue)."* — **the threshold is $5M, not the $2M this page previously recorded**, and the ingested sources are stale on this point. The "cannot be used competitively with our API" clause remains. Some sources additionally describe marker's code as GPL-3.0 and Surya's weights as OpenRAIL-M.

The `datalab-to/chandra` HF card, read the same day, tags **`openrail`** and reports **9B parameters** and olmOCR-Bench **83.1 ± 0.9** with "40+ languages" — Hungarian not among those listed. Note this is the `chandra` repo, not necessarily the Chandra 2 checkpoint described above; verify which artifact you are actually pulling before relying on either number.

For a permissive-only stack this is disqualifying. Chandra 2 is worth reserving as a path for the hardest English pages *if* you clear the revenue threshold; otherwise stay on Apache-2.0 ([[PaddleOCR-VL]], [[olmOCR]], [[dots.ocr]], [[Granite-Docling]]) and MIT ([[Docling]]). See [[Permissive Licensing Constraints]].

## Surya

Datalab's standalone OCR/layout toolkit — OCR, layout, reading order and table detection across many languages, usable independently of marker. Same weights-licensing caveat.

## Related

[[olmOCR]] · [[Docling]] · [[Permissive Licensing Constraints]] · [[olmOCR-Bench]]
