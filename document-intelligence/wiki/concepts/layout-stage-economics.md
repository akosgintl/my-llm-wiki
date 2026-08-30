---
aliases: ["Layout Stage Economics"]
tags: [layout-analysis, serving, cost, optimization]
sources: [ocr-language-models-deployment-critical-pass-2.md, ocr-vdu-complete-study.md, Deep Technical Review - Open-Source OCR - Visual Document Understanding Models - Missed Details and Pipeline Implications.md]
created: 2026-08-27
updated: 2026-08-27
---

# Layout Stage Economics

Only **[[GLM-OCR]]-pipeline and [[PaddleOCR-VL]]** deployments have this decision. [[MinerU]] folds layout into its own VLM; [[DeepSeek-OCR]], [[Baidu Unlimited-OCR]], [[dots.ocr]] and [[HunyuanOCR]] are end-to-end.

## The asymmetry that frames everything

[[PP-DocLayout-V3]] is an RT-DETR-class **detector** — **10–50× cheaper per page than recognition**. It is trivially cheap when placed right and a silent bottleneck when starved.

## Three tiers

| Tier | Placement | ~Latency/page | Economics | When |
|---|---|---|---|---|
| **Minimal (default)** | CPU sidecar with the gateway; ONNX Runtime / OpenVINO INT8; **4–8 real vCPUs** + explicit thread counts | 150–400 ms | 8-vCPU pod ≈ $0.35–0.40/hr → 3–6 pages/s | 24 GB fleet; all batch lanes |
| **Optimal** | shared T4/L4 layout micro-service (batch endpoint, TensorRT) | 15–40 ms | one ~$0.50/hr T4 → 25–60 pages/s, feeding the whole H100 farm | fleet demand > ~5 pages/s sustained |
| **High-end** | same GPU/MIG slice as the VLM (`--layout-device cuda:0`), TensorRT-compiled | 5–15 ms | "free" but spends ~0.5–1 GB of your most expensive VRAM and SM time | interactive lane; monolithic Paddle pipeline; [[RunPod]] (can't rent fractional T4s) |

## The T4 insight

**The compute-capability ≥ 8.0 floor applies to the VLM under [[vLLM]], not to a detector.** The T4s banished from the recognition fleet are exactly right as a shared layout micro-service — and one card feeds a seven-replica H100 farm. See [[MIG and GPU Sharing]].

## Failure modes, ranked

1. **Default Kubernetes CPU requests.** A 2-vCPU layout pod starving an H100 behind it. Give layout real cores and set explicit thread counts.
2. **Rendering the layout decision at 300 DPI** — the same resolution mistake as everywhere else. See [[Rasterization at Model-Native Resolution]].
3. **Co-locating on a 1g.10gb slice**, where the VRAM bite forces `gpu-memory-utilization` down and eats KV headroom.

## Latency accounting

CPU layout adds ~0.2–0.3 s per document versus GPU's ~0.01 s — **meaningful against a 2-second interactive SLO, invisible in batch.** That, not cost, is what usually decides the tier.

## Caveats

These are RT-DETR-class engineering estimates, **not published PP-DocLayout-V3 benchmarks**. The ONNX/OpenVINO export path does exist (community exports validated on 1355 images) — but mind the `mean=0/std=1` normalization gotcha, since ImageNet normalization silently drops detections.

Converted to a common unit in [[Cost per Page Model]]: the layout stage costs **$0.002–0.006 per 1k pages**, 17–130× less than the cheapest recognition tier.

## Related

[[PP-DocLayout-V3]] · [[Two-Stage vs End-to-End Document Parsing]] · [[MIG and GPU Sharing]] · [[Tiered Page Routing]]
