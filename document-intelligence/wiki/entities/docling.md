---
aliases: ["Docling"]
tags: [document-parsing, idp, framework, orchestration]
sources: [Local PDF-Markdown and Document Parsing - State of the Art (August 2026).md, tiered-pdf-pipeline-architecture.md]
created: 2026-08-27
updated: 2026-08-27
---

# Docling

[[IBM Research]]'s document-understanding **framework** — MIT-licensed, donated to the **LF AI & Data Foundation in April 2025**, and the leading open orchestration layer for IDP.

## How the classic pipeline works

Parse the PDF backend (text tokens + rendered bitmap) → layout model → feed detected tables to **TableFormer** (FAST/ACCURATE modes) → match predictions back to the PDF's own cells (language-agnostic, avoiding re-transcription) → assemble a structured `DoclingDocument` exportable to Markdown, HTML or JSON. Handles PDF, DOCX, PPTX, XLSX, HTML, EPUB, images and audio (ASR pipeline). Integrates with LangChain and [[LlamaIndex]].

IBM's Peter Staar (Principal Research Staff Member, IBM Research Zurich; Chair of the Docling Technical Steering Committee at the Linux Foundation) states that avoiding OCR "reduces errors, and it also speeds up the time-to-solution by 30 times."

## Versions and ecosystem (Aug 2026)

`docling` **2.121.0** (2026-08-20); `docling-ibm-models` **3.14.0** (2026-08-11). Supporting packages: docling-serve (Docker/CUDA images), docling-jobkit (Ray/RQ distributed jobs), docling-mcp (agentic/MCP), docling-graph (knowledge graphs), docling-eval. Enterprise integration via Red Hat OpenShift Operator, RHEL AI and OpenShift AI. A single-pass VLM pipeline built on [[Granite-Docling]] is available with `--pipeline vlm`.

## The benchmark weakness

The classic pipeline **scores poorly on [[OmniDocBench]]** — in the dots.ocr paper's Table 1 it posts an overall edit distance of **0.589 (EN) / 0.909 (ZH)**, with near-total failure on formulas (~0.999–1.000 edit) and weak Chinese, against [[MinerU]]'s 0.150/0.357. Part of this reflects OmniDocBench's formatting-sensitive edit-distance metric rather than pure extraction failure — but the gap on formulas and CJK is real. See [[Benchmark Saturation]].

## Where it belongs

Not as the parser, but as the **orchestration and normalization layer**: the `DoclingDocument` schema is the best structured document object model available under MIT, and the format breadth plus RAG integrations are unmatched. Route the actual page parsing to a small specialized VLM. In a [[Tiered Page Routing]] design it is Tier 2 — Docling + TableFormer + Tesseract `hun` for tables and multi-column pages on CPU or a small GPU — with [[PaddleOCR-VL]] behind it on Tier 3.

## Related

[[Granite-Docling]] · [[MinerU]] · [[liteparse]] · [[Tiered Page Routing]] · [[IBM Research]]
