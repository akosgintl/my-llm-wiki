---
aliases: ["Rasterization at Model-Native Resolution"]
tags: [rasterization, performance, pipeline, optimization]
sources: [ocr-rasterize-encoding-bottlenecks.md, ocr-vdu-complete-study.md, ocr-api-call-strategy.md]
created: 2026-08-27
updated: 2026-08-27
---

# Rasterization at Model-Native Resolution

**The single biggest optimization in the whole pipeline, and the most common self-inflicted wound.**

## The mistake

Render at 300 DPI "for quality" (A4 → 2480×3508 px), ship 2–8 MB per page, and let the model's preprocessor immediately downscale to its native 1024–1600 px input. That is **4–9× wasted pixels through the entire chain** — render time, encode time, transfer bytes, server-side decode, resize — **for zero accuracy gain**, because the model never sees the extra pixels.

## Rule 1: render to the model's native input size, not to a DPI

| Model | Target |
|---|---|
| [[DeepSeek-OCR]] / [[Baidu Unlimited-OCR]] base mode | 1024 px long edge (Gundam tiles from 640) |
| [[PaddleOCR-VL]] / [[GLM-OCR]] | dynamic-resolution encoders, but their pipelines crop *regions* — ~1200–1600 px page long edge is plenty |

**Practical target: long edge 1024–1600 px (≈110–150 DPI for A4).** Exceed it only for known tiny-print documents, and then prefer the model's tiling mode over brute DPI.

This alone typically cuts gateway CPU and transfer volume by **75–85%**.

Match the mode exactly where modes are discrete: DeepSeek's Tiny/Small/Base/Large map to 512²/640²/1024²/1280², and the mode also sets the hallucination-risk level — see [[Optical Compression]].

## Timing reality check

200 pages ÷ (16 cores × ~20 pages/s/core) ≈ **0.6–1.5 s** wall-clock at target resolution. Done serially at 300 DPI it is 60–120 s. **Rasterization is only a bottleneck when you do it wrong**; done right it disappears under the GPU time.

## Tooling

Use [[pypdfium2]] (Apache-2.0) or `pdftoppm`/Poppler (GPL, safe across a process boundary). Avoid the AGPL MuPDF family without legal sign-off — see [[Permissive Licensing Constraints]]. `qpdf --split-pages` (<1 s for 200 pages) enables clean per-page CLI parallelism.

**Process pools, never threads** — the GIL plus fitz's thread-unsafety make threading useless; forgetting `ProcessPoolExecutor` costs 10–20×.

```bash
qpdf --split-pages doc.pdf 'pages/pg-%d.pdf'
ls pages/*.pdf | parallel -j "$(nproc)" 'pdftoppm -jpeg -r 135 {} {.}'
```

One caveat on process-per-page with big scanned PDFs: each worker re-opens the file and parses the xref. For a 300 MB scan × 16 workers that is real I/O — give each worker a page range, or use the qpdf split so each touches only its own slice.

## Encoding

**JPEG q90 via libjpeg-turbo is the default** (~120–300 KB/page at 1024 px).

| Choice | Size/page | Verdict |
|---|---|---|
| PNG (zlib-6) | 300–800 KB | ❌ slowest encode, biggest bytes |
| **JPEG q90** | 120–300 KB | ✅ **default** |
| JPEG q75 | 60–150 KB | ringing around small glyphs — measurable CER hit on fine print; previews only |
| Grayscale JPEG q90 | ~40% smaller | ✅ for scan-only corpora (models convert to RGB internally anyway) |
| WebP lossless | ~PNG×0.7 | not worth the encode CPU |

Two server-side reasons JPEG wins beyond size: **[[vLLM]]'s preprocessor must decode every image on CPU before the GPU sees it**, and JPEG decode (SIMD) is several times faster than PNG inflate. At 200 concurrent requests that decode CPU *is* your p99. Verify libjpeg-turbo is actually the backend in both gateway and vLLM images. For heavy resize work use pyvips or OpenCV, not vanilla PIL — SIMD resize is 3–10× faster. Do not gzip JPEG bodies; they are already compressed.

## Related

[[Base64 and Image Transport]] · [[pypdfium2]] · [[Document Fan-Out and Fan-In]] · [[Optical Compression]]
