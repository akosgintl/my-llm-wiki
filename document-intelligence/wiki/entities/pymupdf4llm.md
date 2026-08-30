---
aliases: ["pymupdf4llm"]
tags: [document-parsing, cpu, licensing, tool]
sources: [Local PDF-Markdown and Document Parsing - State of the Art (August 2026).md, ocr-rasterize-encoding-bottlenecks.md, tiered-pdf-pipeline-architecture.md]
created: 2026-08-27
updated: 2026-08-27
---

# pymupdf4llm

Lightweight PyMuPDF extension exposing `to_markdown()`. ~0.01 s/page, no ML, no OCR by default. The best pure-speed default for **born-digital** PDFs feeding RAG — and a licensing trap.

## The AGPL problem

PyMuPDF (and therefore pymupdf4llm) is **AGPL-3.0** or a paid commercial license. This is the single most consequential licensing fact in the local-parsing landscape, because PyMuPDF is everyone's default rasterizer and it propagates:

- The [[DeepSeek-OCR]] toolchain pulls it in, which is the substance of that repo's GitHub Issue #223 disputing its MIT claim.
- Any commercial IDP product that vendors these scripts inherits strong copyleft.

The fix is cheap now and expensive later: standardize on **[[pypdfium2]]** (Python) or **pdftoppm/Poppler** (CLI) and treat the MuPDF family as opt-in-with-legal-review. See [[Permissive Licensing Constraints]].

## Capability limits

Weak on tables, and it **silently returns near-empty output on scans** — a dangerous failure mode, because a page that produces no text looks like a successfully parsed blank page. Any router relying on it needs a text-token-density check. A March 2026 "PyMuPDF-Layout" adds a layout model (DocLayNet F1 ~0.864).

## Related

[[liteparse]] · [[pypdfium2]] · [[Permissive Licensing Constraints]] · [[Tiered Page Routing]]
