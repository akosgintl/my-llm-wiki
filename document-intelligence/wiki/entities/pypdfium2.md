---
aliases: ["pypdfium2"]
tags: [rasterization, cpu, licensing, tool]
sources: [ocr-rasterize-encoding-bottlenecks.md, ocr-vdu-complete-study.md, tiered-pdf-pipeline-architecture.md]
created: 2026-08-27
updated: 2026-08-27
---

# pypdfium2

Python bindings for **PDFium**, Chrome's PDF engine. Apache-2.0/BSD. **The permissive-license workhorse for rasterization** and the recommended default over the faster-but-AGPL MuPDF family.

## Where it sits in the tool matrix

| Tool | License | ~pages/s/core @150 DPI | Note |
|---|---|---|---|
| PyMuPDF (fitz) | **AGPL-3.0** | 20–50 | Fastest Python-native; **not thread-safe** → process pool |
| **pypdfium2 / PDFium** | **Apache-2.0/BSD** | 15–40 | Marginally slower than MuPDF on some docs, irrelevant in parallel |
| pdftoppm (Poppler) | GPL-2 (process boundary = safe) | 10–25 | The classic native Linux answer, trivially parallel via subprocess |
| mutool draw (MuPDF) | AGPL-3.0 | 20–50 | Fastest CLI; AGPL via subprocess of an unmodified binary is *generally* considered safe — get legal sign-off |
| libvips (pdfload) | LGPL | 15–30 | Streams, tiny memory footprint |
| Ghostscript | AGPL-3.0 | 3–10 | Only for color management / PDF repair |
| ImageMagick convert | — | 1–5 | Delegates to Ghostscript with overhead — never for production |
| **qpdf** | Apache-2.0 | 100+ splits/s | Not a rasterizer: splits a 200-page PDF into per-page files in <1 s, enabling clean CLI parallelism |

## Usage pattern

Render to the model's native size, not to a DPI — see [[Rasterization at Model-Native Resolution]]:

```python
import pypdfium2 as pdfium
pdf = pdfium.PdfDocument("doc.pdf")      # open per process
page = pdf[pno]
scale = TARGET_LONG_EDGE / max(page.get_size())
bitmap = page.render(scale=scale)
pil = bitmap.to_pil()                    # then JPEG-encode q90
```

**Process pools, never threads** — the GIL plus fitz's thread-unsafety make threading useless, and forgetting `ProcessPoolExecutor` costs 10–20×. For very large scanned PDFs, give each worker a page *range* (one file open) or use the qpdf split so each worker touches only its own ~1.5 MB slice.

Rust equivalent: `pdfium-render`. Go: `go-pdfium` (cgo→PDFium, permissive) — avoid `go-fitz`, which is AGPL again.

## Related

[[pymupdf4llm]] · [[Rasterization at Model-Native Resolution]] · [[Permissive Licensing Constraints]] · [[liteparse]]
