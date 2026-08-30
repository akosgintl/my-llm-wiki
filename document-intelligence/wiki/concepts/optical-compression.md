---
aliases: ["Optical Compression"]
tags: [architecture, ocr, performance, hallucination]
sources: [Open-Source OCR per Visual Document Understanding Models for Self-Hosted AKS Pipelines - A Comparative Investigation (August 2026).md, ocr-arxiv-github-technical-review.md, Deep Technical Review - Open-Source OCR - Visual Document Understanding Models - Missed Details and Pipeline Implications.md]
created: 2026-08-27
updated: 2026-08-27
---

# Optical Compression

[[DeepSeek AI]]'s thesis: a document page can be encoded into **far fewer vision tokens than its text tokens**, and the accuracy cost of doing so is measurable and predictable.

## The measured curve (Fox benchmark, verbatim)

| Compression ratio (text tokens : vision tokens) | Decoding precision |
|---|---|
| < 10× | **~97%** |
| 10–12× | ~90% |
| 20× | **~60%** |

That curve is the quantitative backbone of the risk model. A 1024² image is 4096 patch tokens compressed 16× down to 256.

## Why it is an operational setting, not a research curiosity

Two consequences follow directly:

**1. Dense pages need Base/Large/Gundam, not Small.** Accuracy degrades with page *text density* per mode, so the mode must match the document, and your raster target must match the mode's native size exactly. See [[Rasterization at Model-Native Resolution]].

**2. Fewer visual tokens means more hallucination.** The critique paper found that **lower visual-token counts correlate with more reliance on language priors**. So the compression setting is simultaneously a hallucination-risk setting, and the confidence model should **weight low-visual-token outputs as higher risk**. See [[Linguistic Crutch and Faithfulness]] and [[Confidence Engineering]].

## Throughput is the payoff

~200,000 pages/day on a single A100-40G, driven by the MoE decoder (~570M active of 3B) plus the compression. That is what makes DeepSeek-OCR the pages-per-dollar choice for bulk digitization of clean digital PDFs — with a retry budget for [[Repetition Loops in VLM OCR]].

## Where the idea went next

- **DeepSeek-OCR 2 / DeepEncoder V2** keeps the budget (256–1120 tokens, k×144+256) but adds learned reading-order reordering, matching Gemini-3-Pro's token budget.
- **[[Baidu Unlimited-OCR]]** took the same encoder and attacked the *output* side instead, bounding KV growth with [[R-SWA Reference Sliding Window Attention]].
- **[[Qwen Model Family]]** exposes the same idea as a tunable cost knob: ~32× spatial compression with a steerable 256–1280 token budget per image. For region crops, cap it low.

## Related

[[DeepSeek-OCR]] · [[Rasterization at Model-Native Resolution]] · [[Linguistic Crutch and Faithfulness]] · [[R-SWA Reference Sliding Window Attention]]
