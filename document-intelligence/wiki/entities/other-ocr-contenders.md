---
aliases: ["Other OCR Contenders"]
tags: [ocr, vlm, document-parsing, model]
sources: [Open-Source OCR per Visual Document Understanding Models for Self-Hosted AKS Pipelines - A Comparative Investigation (August 2026).md, ocr-vdu-complete-study.md, Local PDF-Markdown and Document Parsing - State of the Art (August 2026).md]
created: 2026-08-27
updated: 2026-08-27
---

# Other OCR Contenders

Models tracked but not shortlisted, kept here so the landscape stays complete without proliferating thin pages. The field ships a new SOTA claim roughly monthly — re-validate quarterly.

## Worth watching

- **Nanonets-OCR2 / OCR-3.** **Corrected 2026-08-27 — the strong numbers are not the open weights.** The `nanonets/` org publishes only **OCR-s** (Jun 2025), **OCR2-3B** (Oct 2025) and **OCR2-1.5B-exp** (Dec 2025); **there is no OCR-3 checkpoint**, and OCR2-Plus is served through the Docstrange API. The open `Nanonets-OCR2-3B` card (base Qwen2.5-VL-3B) reports **olmOCR-bench 69.5** and MDPBench 64.2 — far below the 87.4 and 82.0 the leaderboard shows for the hosted products — and names 11 languages (en, zh, fr, es, pt, de, it, ru, ja, ko, ar) with **Hungarian absent**. Semantic tagging and KIE remain the line's distinguishing feature.
- **NVIDIA Nemotron-OCR-v2** (Apr 2026, ~84M). Extreme speed: 34.7 pages/s multilingual on A100 (~28× PaddleOCR v5), TensorRT/NIM, 6 languages. Recognition-focused, not full document parsing. **Nemotron Parse 2.0** (0.9B, 2026-08-03) is a multilingual, chart-aware alternative running on Ampere. **Checked 2026-08-27:** it is licensed under **OpenMDW-1.1 together with the NVIDIA Open Model License Agreement** (tokenizer CC-BY-4.0) — not a standard permissive licence, so it needs the same deliberate decision as [[HunyuanOCR]]. Its v2.0 vocabulary expansion (52,329 → 72,256 tokens) is evaluated on **IndicVisionBench** (10 Indic languages) and MOSCAR; **Hungarian is not named**, and the multilingual investment is visibly aimed at Indic scripts.
- **Qianfan-OCR** (arXiv 2603.13398). Its Table 6 is the definitive end-to-end versus two-stage comparison and the citation behind [[The OCR-to-Text Boundary Limit]].
- **MonkeyOCR-Pro · FireRed-OCR · Youtu-Parsing · InternVL** — various niches, no distinguishing evidence in this corpus.

## Managed APIs (comparison / fallback only)

- **Mistral OCR 4** — bounding boxes, typed blocks, ~$4 per 1000 pages, strong on printed text and tables, and notably **exposes per-word confidence via API** — which none of the self-hosted VLMs do. Not self-hostable; useful as a comparison point and hard-case overflow if compliance permits.
- **Gemini 3 / 3.1 Pro and Flash-Lite** — still lead independent multi-script evaluation by a wide margin. See [[Multi-Script OCR Benchmarks]].

## Superseded

- **PP-StructureV3** — the classic modular PaddleOCR pipeline, OmniDocBench v1.5 overall 86.73. Solid CPU/GPU option, superseded in accuracy by [[PaddleOCR-VL]].
- **GOT-OCR2** (2024) — now dated ([[olmOCR-Bench]] 48.3).
- **Nougat** (Meta, 2023) — Donut-based academic-PDF model, still installable as `nougat-ocr` but **effectively unmaintained** (dependency/transformers breakage, open issues), English/arXiv-only. Not recommended for new pipelines.
- **Unstructured** — general ingestion/chunking library, convenient for heterogeneous enterprise formats, but parsing fidelity on complex tables and formulas trails the specialized VLMs. Use as an orchestration layer, not a parser.

## Related

[[PaddleOCR-VL]] · [[HunyuanOCR]] · [[Marker and Chandra]] · [[Benchmark Saturation]]
