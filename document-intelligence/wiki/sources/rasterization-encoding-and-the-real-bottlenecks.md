---
aliases: ["Rasterization Encoding and the Real Bottlenecks"]
tags: [rasterization, performance, licensing, optimization, source]
sources: [ocr-rasterize-encoding-bottlenecks.md]
created: 2026-08-27
updated: 2026-08-27
---

# Rasterization Encoding and the Real Bottlenecks

**Source:** ocr-rasterize-encoding-bottlenecks.md
**Date ingested:** 2026-08-27
**Type:** critical analysis
**Position in the corpus:** fourth document (2026-08-24 21:21) — a deliberately critical pass over the upstream half of the pipeline.

## Summary

Opens with the claim that most OCR pipelines rasterize at the wrong resolution and then argue about base64. Covers the rasterization tool matrix with licensing, output format and encoding, an honest accounting of base64, a ranked ten-item bottleneck audit, and a gateway-language comparison.

## Key Claims

- **Rule 1: render to the model's native input size, not to a DPI.** Rendering at 300 DPI when the model downscales to ~1024 px is **4–9× wasted pixels through the entire chain for zero accuracy gain**. Target long edge 1024–1600 px (≈110–150 DPI for A4). **This alone cuts gateway CPU and transfer volume by ~75–85%.**
- **Rasterization is only a bottleneck when done wrong** — 200 pages in 0.6–1.5 s on 16 cores at target resolution, versus 60–120 s serially at 300 DPI.
- **The licensing critique you should not skip:** PyMuPDF is AGPL-3.0 and is everyone's default. Standardize on [[pypdfium2]] or `pdftoppm`. "A cheap decision now and an expensive one later."
- **Process pools, never threads** — the GIL plus fitz's thread-unsafety make threading useless; forgetting `ProcessPoolExecutor` costs 10–20×.
- **JPEG q90 via libjpeg-turbo is the default**, partly because **vLLM's frontend must decode every image on CPU** and JPEG decode is several times faster than PNG inflate. At 200 concurrent requests that decode CPU is your p99.
- **Base64 encoding is never the bottleneck** (~1–2 GB/s; ~40 ms for 200 pages). The real costs are +33% wire (matters over WAN to RunPod, not in-cluster), `json.dumps` copying megabyte strings (use orjson), vLLM frontend buffering, and memory churn.
- **`file://` with a shared `emptyDir` is the best in-cluster escape hatch** — but as a *sidecar* pattern only; cross-node RWX file passing costs more than base64 saved.
- **Gateway language:** Rust (tokio + rayon, no GIL) is the best raw fit; Go is 80% of the benefit at 30% of the effort; **.NET 8 is a genuine argument on AKS** because of first-class Azure SDKs. Hybrid, not rewrite — keep Python where the ML ecosystem lives.

## Entities Mentioned

- [[pypdfium2]] — the permissive workhorse; the tool matrix
- [[pymupdf4llm]] — fastest but AGPL
- [[vLLM]] — the frontend CPU bottleneck
- [[RunPod]] — where the +33% wire cost becomes real
- [[neural-maze production-ocr-course]] — the Rust/Axum gateway precedent
- [[DeepSeek-OCR]] — its toolchain drags in AGPL PyMuPDF

## Concepts Covered

- [[Rasterization at Model-Native Resolution]] — Rule 1, the tools, the encoding matrix
- [[Base64 and Image Transport]] — the honest accounting and the ten-item bottleneck audit
- [[Permissive Licensing Constraints]] — the AGPL propagation problem
- [[Document Fan-Out and Fan-In]] — straggler pages, event-loop blocking

## Caveats stated by the source

Numbers marked with a tilde are ballpark engineering figures — benchmark on your own document mix before hardcoding. AGPL via subprocess of an unmodified binary is "generally considered safe, but get legal sign-off."
