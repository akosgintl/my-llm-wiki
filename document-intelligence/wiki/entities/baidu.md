---
aliases: ["Baidu"]
tags: [organization]
sources: [Open-Source OCR per Visual Document Understanding Models for Self-Hosted AKS Pipelines - A Comparative Investigation (August 2026).md, ocr-arxiv-github-technical-review.md, Deep Technical Review - Open-Source OCR - Visual Document Understanding Models - Missed Details and Pipeline Implications.md, Local PDF-Markdown and Document Parsing - State of the Art (August 2026).md]
created: 2026-08-27
updated: 2026-08-27
---

# Baidu

The most prolific organization in this landscape, and the only one shipping across every layer of the stack.

## What they ship

| Component | Role | License |
|---|---|---|
| [[PaddleOCR-VL]] | 0.9B multilingual document VLM; Hungarian confirmed | Apache-2.0 |
| [[PP-DocLayout-V3]] | layout detector used by *both* PaddleOCR-VL and [[GLM-OCR]] | Apache-2.0 |
| [[PP-OCRv6]] | 34.5M CTC recognizer; hallucination-free cross-check | Apache-2.0 |
| [[Baidu Unlimited-OCR]] | R-SWA long-context one-shot parser | MIT |
| ERNIE-4.5-0.3B | the decoder inside PaddleOCR-VL | — |
| FastDeploy | their fastest serving backend (~2.0 pages/s A100) | — |
| Qianfan-OCR | the end-to-end vs two-stage comparison study | — |

## The strategic read

Baidu occupies both ends of the design space simultaneously: PaddleOCR-VL is the two-stage, layout-first, maximum-multilingual arm, while Unlimited-OCR is a continue-trained [[DeepSeek-OCR]] taken in the opposite direction — end-to-end, one-shot, long-context. They are not competing products; they are different answers to different workloads, and a pipeline can reasonably run both.

Their permissive licensing (Apache-2.0 / MIT throughout) is a genuine practical advantage over [[Marker and Chandra]]'s OpenRAIL-M caps and [[HunyuanOCR]]'s NOASSERTION. See [[Permissive Licensing Constraints]].

The consistent caveat: **their published benchmark numbers are vendor-reported**, and their models are among those most exposed by independent multi-script evaluation — PaddleOCR-VL holds up well there, but the leaderboard positions do not transfer. See [[Benchmark Saturation]].

## Related

[[PaddleOCR-VL]] · [[Baidu Unlimited-OCR]] · [[PP-DocLayout-V3]] · [[PP-OCRv6]]
