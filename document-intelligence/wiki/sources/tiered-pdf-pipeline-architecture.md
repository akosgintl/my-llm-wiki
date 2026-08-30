---
aliases: ["Tiered PDF Pipeline Architecture"]
tags: [architecture, pipeline, hungarian, licensing, source]
sources: [tiered-pdf-pipeline-architecture.md]
created: 2026-08-27
updated: 2026-08-27
---

# Tiered PDF Pipeline Architecture

**Source:** tiered-pdf-pipeline-architecture.md
**Date ingested:** 2026-08-27
**Type:** reference architecture
**Position in the corpus:** tenth document (2026-08-26 10:53) — the shortest and most concrete. It converts the surveys into a buildable design.

## Summary

A self-hosted, permissive-license-only, pluggable, Hungarian-first PDF→Markdown reference architecture. Three parsing tiers with a per-page router, a shared block schema, Hungarian-aware validation, an escalation loop, and a four-stage rollout plan. Contains Mermaid diagrams of both the pipeline and the router decision signals.

## Key Claims

- **Three tiers, routed per page:** Tier 1 fast text ([[liteparse]], CPU, ~ms/page) · Tier 2 layout pipeline ([[Docling]] + TableFormer + [[Tesseract]] `hun`) · Tier 3 full-page VLM ([[PaddleOCR-VL]] via [[vLLM]]).
- **The router uses five signals at <1 ms on CPU** — text token density, bitmap coverage ≥0.95, embedded fonts (`GlyphlessFont` indicates prior OCR), CID-escape ratio >0.5, and **hu_HU dictionary ratio + mojibake/Latin-2 check** — with a "fail 2 of 5" voting rule.
- **Failed validation escalates to the next tier up**, so a bad Tier-1 parse costs one retry rather than a corrupted document.
- **All tiers normalize to one `Block` schema** carrying `confidence` and `source_tier` — the latter from day one, because it is what makes the escalation matrix measurable.
- **Every slot is swappable behind one `Parser` protocol**, and Tier 3 hides behind an OpenAI-compatible endpoint, so changing the VLM is a config change.
- **Validation must be Hungarian-aware** — an English wordlist would flag every correct Hungarian page as garbage and escalate the whole corpus to the GPU.
- **Legacy encodings route straight to OCR**: older Hungarian PDFs often carry broken Latin-2/CP-1250 text layers with `õ` for `ő`, which should skip Tier 1 entirely.
- **Tesseract `hun` handles the double acute well on clean scans** — it is VLM output on *degraded* scans where drift appears.
- **Primary tuning signal: the escalation matrix.** Escalation above ~30% means miscalibrated thresholds or a scan-heavier corpus than assumed.

## Explicit licensing exclusions

| Excluded | Reason | Replacement |
|---|---|---|
| PyMuPDF / pymupdf4llm | AGPL-3.0 | [[pypdfium2]] + liteparse |
| Marker | GPL-3.0 code + OpenRAIL-M weights | Docling, PaddleOCR-VL |
| Surya weights, DocLayout-YOLO | OpenRAIL-M / AGPL | [[PP-DocLayout-V3]] |
| [[olmOCR]] | Apache but English-focused | PaddleOCR-VL |
| [[MinerU]] | permissive-ish with attribution conditions | optional |

## Entities Mentioned

[[liteparse]] · [[Docling]] · [[Tesseract]] · [[PaddleOCR-VL]] · [[pypdfium2]] · [[PP-DocLayout-V3]] · [[dots.ocr]] · [[Granite-Docling]] · [[vLLM]] · [[pymupdf4llm]] · [[Marker and Chandra]]

## Concepts Covered

[[Tiered Page Routing]] — this document *is* the concept's primary source · [[Hungarian OCR and the Double Acute Test]] · [[Permissive Licensing Constraints]] · [[Pipeline as Platform, Model as Config]]

## Note

This document assumes a **different serving posture** from the AKS study: Celery/RQ + vLLM rather than Service Bus + KEDA + MIG, and PaddleOCR-VL rather than Qwen3.5-4B behind the glmocr SDK. The two are not in conflict — this is the CPU-first, permissive-only, single-GPU shape of the same problem, and its Tier-3 slot is exactly where the AKS study's model bake-off plugs in.
