---
aliases: ["PP-DocLayout-V3"]
tags: [layout-analysis, detector, cpu, tool]
sources: [ocr-language-models-deployment-critical-pass-2.md, ocr-vdu-complete-study.md, ocr-arxiv-github-technical-review.md, Deep Technical Review - Open-Source OCR - Visual Document Understanding Models - Missed Details and Pipeline Implications.md, tiered-pdf-pipeline-architecture.md]
created: 2026-08-27
updated: 2026-08-27
---

# PP-DocLayout-V3

[[Baidu]]'s layout detector, Apache-2.0 (`PaddlePaddle/PP-DocLayoutV3`). The front stage of both [[GLM-OCR]] and [[PaddleOCR-VL]], and the reusable component of the [[glmocr SDK]].

## What it produces

DETR/RT-DETR-class detector with a PPHGNetV2-L backbone, **25 categories**, instance segmentation for skew and curve robustness, and **multi-point/polygonal localization** rather than plain rectangles. Crucially it also has a **reading-order prediction branch** using a Global-Pointer-style pointer mechanism over detected regions.

That last point is easy to miss and worth acting on: **the layout stage outputs order, not just boxes** — the assembly step should consume it rather than re-sorting regions by y-coordinate.

Its boxes also carry detection confidence scores, which are the free first signal in the [[Confidence Engineering]] stack.

## Why it is cheap

It is a *detector*, so it costs 10–50× less per page than recognition. It is trivially cheap when placed right and a silent bottleneck when starved of CPU. See [[Layout Stage Economics]].

## The ONNX path (and its gotcha)

Community ONNX exports enable a **Paddle-free CPU layout tier**: `alex-dinh/PP-DocLayoutV3-ONNX` (via Paddle2ONNX PR #1619) and `AlexTransformer/PP-DocLayoutV3-onnx` (MIT, validated against the Paddle native pipeline on 1355 images).

**Critical gotcha:** the ONNX model expects `mean=[0,0,0], std=[1,1,1]` — i.e. rescale by 1/255 only, **not ImageNet normalization**. Wrong normalization drops detections (13→12 in the validation run). The native torch source also hardcodes batch=1, which blocks naive ONNX batching unless patched; HF `PaddlePaddle/PP-DocLayoutV3_safetensors` supports batched inference via the transformers `object-detection` pipeline. OpenVINO path via `paddleocr_vl_ov` (add_layout branch).

Earlier material treated "does a V3 ONNX/OpenVINO export exist?" as an unverified prerequisite for the CPU tier. It does — with the normalization caveat above.

## Related

[[GLM-OCR]] · [[PaddleOCR-VL]] · [[glmocr SDK]] · [[Layout Stage Economics]] · [[Tiered Page Routing]]
