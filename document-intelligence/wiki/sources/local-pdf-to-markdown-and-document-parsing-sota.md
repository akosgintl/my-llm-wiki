---
aliases: ["Local PDF to Markdown and Document Parsing State of the Art"]
tags: [document-parsing, idp, licensing, benchmarks, source]
sources: [Local PDF-Markdown and Document Parsing - State of the Art (August 2026).md]
created: 2026-08-27
updated: 2026-08-27
---

# Local PDF to Markdown and Document Parsing State of the Art

**Source:** Local PDF-Markdown and Document Parsing - State of the Art (August 2026).md
**Date ingested:** 2026-08-27
**Type:** survey
**Position in the corpus:** ninth document (2026-08-25 22:40) — widens the frame from *OCR models* to the whole *local parsing ecosystem*, including CPU tools and orchestration frameworks the earlier documents skipped.

## Summary

Surveys local PDF→Markdown parsing across VLMs, orchestration frameworks and heuristic extractors, with head-to-head comparison on tables, math, reading order, scans, speed, hardware and licensing, plus scenario-based recommendations.

## Key Claims

- **The center of gravity has shifted from CPU pipelines to small, self-hostable VLMs.** Sub-2B document-specialized models are now SOTA: a 0.9B [[PaddleOCR-VL]] and 1.2B [[MinerU]] both surpass Gemini 2.5 Pro, GPT-4o and Qwen2.5-VL-72B on OmniDocBench.
- **[[Docling]] is the leading open orchestration framework**, now a Linux Foundation project (donated April 2025). Latest `docling` 2.121.0 (2026-08-20).
- **"liteparse" is a real tool**: `run-llama/liteparse`, the Apache-2.0, Rust-based engine behind LlamaParse — a fast layout-aware **heuristic** extractor, a peer of [[pymupdf4llm]], not of Docling/MinerU/marker.
- **Licensing is the biggest practical differentiator.** MIT (Docling) and Apache-2.0 (PaddleOCR-VL, liteparse, MinerU code, olmOCR, dots.ocr, Granite-Docling) are clean; [[Marker and Chandra]] weights use a modified OpenRAIL-M with a <$2M revenue/funding cap; pymupdf4llm inherits AGPL-3.0.
- **Both dominant benchmark families are near saturation.** LlamaIndex has publicly called [[OmniDocBench]] "saturated"; [[olmOCR-Bench]]'s header/footer category can reward *omitting* content.
- **The recommended default stack:** Docling as the orchestration/normalization layer, with page parsing routed to a small specialized VLM — PaddleOCR-VL for clean commercial licensing, MinerU where you want its mature tooling and CJK strength.
- **The architecture pattern is tiered routing**: digital text → liteparse/pymupdf4llm; structured/tabular → Docling+TableFormer or a small VLM; scanned/complex/math → full-page VLM. This mirrors what marker and MinerU already do internally.
- **"Old Scans Math" remains hard for everyone** — top scores under 90, most 40–55.

## Entities Mentioned

- [[Docling]] and [[Granite-Docling]] — the orchestration layer and its 258M VLM
- [[liteparse]], [[pymupdf4llm]] — the heuristic/CPU tier
- [[MinerU]], [[PaddleOCR-VL]], [[dots.ocr]], [[olmOCR]], [[DeepSeek-OCR]] — the VLM tier
- [[Marker and Chandra]] — strong but licence-blocked
- [[Other OCR Contenders]] — Nanonets OCR-3, PP-StructureV3, GOT-OCR2, Nougat, Unstructured, Surya
- [[IBM Research]] — Docling's steward
- [[OmniDocBench]], [[olmOCR-Bench]] — the two benchmark families

## Concepts Covered

- [[Tiered Page Routing]] — named here as the architecture pattern
- [[Benchmark Saturation]] — including the metric-artifact argument
- [[Permissive Licensing Constraints]] — OpenRAIL-M caps and AGPL

## Caveats stated by the source

Benchmarks are saturating and gameable; treat 1–2 point deltas as noise and run your own corpus. Throughput numbers are best-case on datacenter GPUs. The field moves monthly — architect for swappable parser backends. **A source-quality warning:** one vendor page (idp-software.com) misdated Granite-Docling and mischaracterized the LF donation; primary sources confirm 2025-09-17 and April 2025.
