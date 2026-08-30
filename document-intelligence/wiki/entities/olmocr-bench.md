---
aliases: ["olmOCR-Bench"]
tags: [benchmarks, evaluation, document-parsing]
sources: [Local PDF-Markdown and Document Parsing - State of the Art (August 2026).md, Open-Source OCR per Visual Document Understanding Models for Self-Hosted AKS Pipelines - A Comparative Investigation (August 2026).md, ocr-arxiv-github-technical-review.md, Deep Technical Review - Open-Source OCR - Visual Document Understanding Models - Missed Details and Pipeline Implications.md]
created: 2026-08-27
updated: 2026-08-27
---

# olmOCR-Bench

[[Allen Institute for AI]]'s unit-test-style benchmark, and the second of the two dominant benchmark families alongside [[OmniDocBench]].

Per its dataset card: *"a dataset of 1,403 PDF files, plus 7,010 unit test cases that capture properties of the output that a good OCR system should have"* — pass/fail rather than edit distance, on predominantly **English** documents.

## Scores

**Availability column added 2026-08-27.** The table previously mixed downloadable weights with hosted products in one ranking, which made the leaderboard read as if the top entries were self-hostable. **They are not.**

| System | Score | Weights |
|---|---|---|
| Nanonets OCR-3 | 87.4 (leads the Nanonets IDP leaderboard; Unsiloed AI publicly claims 88.0 as of May 2026) | ❌ **hosted only** — no OCR-3 checkpoint exists under `nanonets/` |
| Chandra 2 | 85.8–85.9 | ⚠️ **restricted** — modified AI Pubs Open Rail-M, free only under $5M funding/revenue |
| [[LightOnOCR]]-2-1B | 83.2 ± 0.9 | ✅ Apache-2.0 |
| Datalab Marker | 83.2 (#2 on the Nanonets leaderboard) | ⚠️ **restricted** — same Datalab weights licence |
| [[olmOCR]] 2 | 82.4 | ✅ Apache-2.0 |
| Nanonets OCR2+ | 82.0 | ❌ **hosted** — served via the Docstrange API. The open `nanonets/Nanonets-OCR2-3B` scores **69.5** on this benchmark |
| [[PaddleOCR-VL]] | ~80.0 | ✅ Apache-2.0 |
| [[dots.ocr]] | ~79.1 (v1.5 ~83.9) | ✅ MIT |
| DeepSeek-OCR 2 | ~76.3 | ✅ repo Apache-2.0 (AGPL toolchain wrinkle) |
| Marker 2 | 76.0 overall / 83.5 born-digital | ⚠️ **restricted** |
| [[MinerU]] | 75.8 | ⚠️ custom Apache-based |
| DeepSeek-OCR | ~75.4–75.7 | ⚠️ MIT disputed (AGPL dep) |
| [[GLM-OCR]] | ~75.2 (**Long/Tiny-text only ~35.7**) | ✅ MIT |
| GOT-OCR2 | 48.3 | ✅ open, but dated (2024) |
| [[liteparse]] | 39.1 (source reports 0.391 on a 0-1 scale) | ✅ Apache-2.0 |

## Reading it critically

- **English-only** — of limited value for a Hungarian corpus.
- **It is olmOCR's own benchmark**, so read its placement of competitors accordingly; the same caution applies to the Nanonets-hosted leaderboard for Nanonets models.
- **The top of this table is not self-hostable.** Of the four systems above 83, one is hosted-only and two carry a revenue-capped weights licence. Under a permissive-only rule the effective open leader here is [[LightOnOCR]]-2-1B at 83.2 — which declares 11 languages and no Hungarian. See [[Coverage and Exclusion Register]].
- Its **header/footer category can reward *omitting* content**, which is a real scoring artifact.
- "Old Scans Math" remains hard for everyone — top scores under 90, most in the 40–55 range.

GLM-OCR's ~35.7 on Long/Tiny text is the single most actionable number here: it is the concrete shape of that model's documented weakness.

## Related

[[OmniDocBench]] · [[olmOCR]] · [[Marker and Chandra]] · [[Benchmark Saturation]]
