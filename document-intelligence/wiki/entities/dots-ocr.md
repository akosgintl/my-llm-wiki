---
aliases: ["dots.ocr"]
tags: [ocr, vlm, document-parsing, multilingual, model]
sources: [Open-Source OCR per Visual Document Understanding Models for Self-Hosted AKS Pipelines - A Comparative Investigation (August 2026).md, ocr-serving-recipes-runpod-v2.md, ocr-language-models-deployment-critical-pass-2.md, ocr-vdu-complete-study.md, ocr-arxiv-github-technical-review.md, Deep Technical Review - Open-Source OCR - Visual Document Understanding Models - Missed Details and Pipeline Implications.md, Local PDF-Markdown and Document Parsing - State of the Art (August 2026).md]
created: 2026-08-27
updated: 2026-08-27
---

# dots.ocr

1.7B single-VLM parser from rednote-hilab (Xiaohongshu), MIT-licensed, built on the **Qwen2.5-1.5B** foundation. Renamed **dots.mocr / v1.5** in 2026, which adds SVG parsing of charts and graphics — a structured-chart route no other candidate offers.

## The independent-benchmark champion

Vendor OmniDocBench is only ~88.4, well below the leaders — but on evaluations that are not vendor-run it is the strongest open model:

- [[OCR Arena]] blind ELO: **#12 (ELO 1442)**, beating [[GLM-OCR]] with an 88.9% win rate across 9 matchups.
- socOCRbench multi-script: ~0.478, best of the open models.
- GlotOCR mid-resource scripts: second overall.
- MORE multilingual: 84.31.
- [[olmOCR-Bench]]: ~79.1 (later v1.5 builds ~83.9).

This inversion versus the vendor leaderboard is the canonical illustration of [[Benchmark Saturation]].

## Prompt-switchable modes

One model, four tasks selected by prompt: full layout+recognition JSON (`prompt_layout_all_en`), **layout-only detection** (`prompt_layout_only_en`), recognition-only, and grounding. The layout-only mode is a cheap **second-opinion layout signal** for spatial-divergence features in the [[Confidence Engineering]] stack.

## Documented limitations

- **Endless repetition on runs of continuous special characters** — ellipses and underscores. That is exactly dotted table-of-contents leader lines and form blanks, i.e. standard Hungarian invoice and form furniture. Pre-strip or regex-guard them. See [[Repetition Loops in VLM OCR]].
- **In-document pictures are not parsed at all.**
- **Incomplete page transcription**: an independent French benchmark found it skips text-dense regions it mistakes for pictures on colorful layouts.
- **The local model directory name must contain no periods** (`DotsOCR`, not `dots.ocr`) — a temporary transformers-integration workaround.
- Slow: ~0.10 images/s, ~0.35 pages/s on A100; VRAM balloons at high batch (~78.5 GB measured in one high-batch run). Treat `max_num_seqs` as a hard ceiling (4 on 24 GB).

Handles up to 11,289,600 pixels; downsample or raise DPI to 200 above that. Officially integrated in [[vLLM]] since v0.11.0 (published evals used 0.9.1); the transformers path is markedly slower.

## Verdict

Buy it for accuracy and reading order on structured multilingual documents and forms where per-region JSON plus bounding boxes are needed, and where throughput is secondary. It is a serious Hungarian candidate on multi-script evidence. See [[Hungarian OCR and the Double Acute Test]].

## Related

[[PaddleOCR-VL]] · [[HunyuanOCR]] · [[GLM-OCR]] · [[OCR Arena]] · [[Multi-Script OCR Benchmarks]]
