---
aliases: ["MIG and GPU Sharing"]
tags: [serving, infrastructure, gpu, optimization]
sources: [ocr-api-call-strategy.md, ocr-vdu-complete-study.md, ocr-serving-recipes-runpod-v2.md]
created: 2026-08-27
updated: 2026-08-27
---

# MIG and GPU Sharing

A 0.9–4B document model **cannot saturate an H100**. Replicas are the win, and how you slice the card decides whether that works.

## The three mechanisms

| Mechanism | Isolation | Memory protection | Latency | Verdict |
|---|---|---|---|---|
| **MIG** | hardware (SM + memory slices) | **yes — hard** | deterministic per slice | ✅ **production default** (H100/A100) |
| MPS | process co-scheduling on shared SMs | **no** — one OOM or crash can take down neighbors | good utilization, noisy-neighbor risk | only for squeezing 0.9B replicas onto non-MIG cards |
| Time-slicing | none (context switch) | no | worst p99 | ❌ never for inference |

## MIG profiles (H100-80GB)

`1g.10gb × 7` · `2g.20gb × 3` · `3g.40gb × 2` · `7g.80gb × 1`

One vLLM process per slice (`CUDA_VISIBLE_DEVICES=MIG-<uuid>`); on AKS via the GPU Operator's `mig.strategy`.

## Allocation per model

| Model | H100 strategy |
|---|---|
| [[GLM-OCR]] / Qwen3.5-4B slot | **7 × 1g.10gb replicas** — the small model can't saturate the card |
| **Qwen3.5-9B** (recognition default since 2026-08-27) | **2 × 3g.40gb**, or 3 × 2g.20gb in FP8. Promoting the default from 4B to 9B is a MIG-profile change, not a hardware change — but it cuts replica count roughly in half, so re-derive concurrency from [[vLLM Continuous Batching and Concurrency Sizing]] rather than reusing the 4B numbers |
| [[PaddleOCR-VL]] | 3 × 2g.20gb; don't raise context — scale the **CPU layout workers** instead |
| [[DeepSeek-OCR]] | full card at seqs 256 (bulk) or 2 × 3g.40gb; 20 GB slices leave thin KV headroom for 3B MoEs |
| **[[Baidu Unlimited-OCR]]** | **full GPU, no MIG** — a 40-page prefill wants the whole card; context *is* the product |
| [[dots.ocr]] | 2 × 3g.40gb only; converts capacity into batch, never into speed |
| [[MinerU]] | 3 × 2g.20gb + mineru-router |

Mixed fleet: 1 × 3g.40gb (dots or DeepSeek) + 4 × 1g.10gb (small-model farm) on one card. Unlimited-OCR always un-sliced.

## The 24 GB fleet (A10 / L4 / RTX 4090)

**These cards cannot MIG.** Run **one vLLM process per card**. Two processes at `--gpu-memory-utilization 0.45` each is possible but couples their failure domains for little gain on 3B models. Only GLM-OCR is small enough to consider it, and **MPS + 2 × 0.9B on an L4 is the only defensible combo**.

The common practical minimum is **24 GB, compute capability ≥ 8.0 (Ampere+)** — forced by PaddleOCR-VL's Ampere floor and dots.ocr's batch memory ballooning. On Azure/AKS that means A10 (no L4 on Azure); on [[RunPod]], RTX 4090; elsewhere, L4.

**[[RunPod]] exposes no MIG at all** — rent the smaller GPU per replica instead of packing one card.

## The one place T4s belong

The compute-capability ≥ 8.0 floor applies to the **VLM under vLLM**, not to a detector. A shared T4 running [[PP-DocLayout-V3]] as a layout micro-service feeds an entire 7-replica H100 farm. The banished T4s are exactly right there. See [[Layout Stage Economics]].

## Caution for Qwen3.5

Its hybrid **Gated DeltaNet** attention (~75% linear-attention layers) changes KV-cache math. **Re-validate MIG slice memory profiles before fixing slice sizes** — the standard KV heuristics assume full attention. See [[Qwen Model Family]].

## Related

[[Parallelism Stack for OCR Serving]] · [[Load Balancing Inference Pools]] · [[Layout Stage Economics]] · [[vLLM]]
