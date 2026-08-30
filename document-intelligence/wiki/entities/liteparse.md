---
aliases: ["liteparse"]
tags: [document-parsing, cpu, throughput, tool]
sources: [Local PDF-Markdown and Document Parsing - State of the Art (August 2026).md, tiered-pdf-pipeline-architecture.md]
created: 2026-08-27
updated: 2026-08-27
---

# liteparse

`run-llama/liteparse` — the open-sourced (Apache-2.0) core processing engine behind LlamaParse, from [[LlamaIndex]]. PyPI `liteparse`, npm `@llamaindex/liteparse`.

## What it actually is

Written in **Rust** with Python (PyO3), Node (napi-rs) and WASM bindings. Uses a custom **PDFium fork** plus a grid-projection algorithm. Runs **entirely locally with no LLMs or API keys**; emits Markdown, JSON or text with bounding boxes; OCR available via Tesseract.

It is a fast, layout-aware **heuristic** text extractor, not a VLM. That places it as a peer of [[pymupdf4llm]], not of [[Docling]], [[MinerU]] or [[Marker and Chandra]].

## Benchmarks

| Benchmark | Score |
|---|---|
| opendataloader-bench | 0.875 |
| [[olmOCR-Bench]] | 0.391 |
| ParseBench | 0.3279 |

Top scores among *model-free/heuristic* tools, far below VLM parsers. With OCR off it reaches ~1,721 pages/sec but "collapses on non-linear layouts since it has no layout model."

LlamaIndex's own positioning: *"If you're building an agent…that needs to quickly read a PDF and move on, use LiteParse. If you're building a document intelligence product that needs to nail every table, use LlamaParse."*

## Where it belongs

**Tier 1 of a [[Tiered Page Routing]] pipeline** — clean digital text only, CPU, ~ms/page. Its Apache-2.0 licence makes it the permissive replacement for [[pymupdf4llm]] (AGPL-3.0) in a commercial stack. Use it as a cheap CPU pre-filter to skip already-clean pages before spending GPU time on hard ones.

## Related

[[pymupdf4llm]] · [[pypdfium2]] · [[Docling]] · [[Tiered Page Routing]] · [[Permissive Licensing Constraints]]
