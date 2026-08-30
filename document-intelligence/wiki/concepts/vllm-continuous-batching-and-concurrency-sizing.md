---
aliases: ["vLLM Continuous Batching and Concurrency Sizing"]
tags: [serving, performance, latency, optimization]
sources: [ocr-api-call-strategy.md, ocr-vdu-complete-study.md, ocr-serving-recipes-runpod-v2.md]
created: 2026-08-27
updated: 2026-08-27
---

# vLLM Continuous Batching and Concurrency Sizing

**The server is the batcher.** With [[vLLM]] continuous batching you never build image batches client-side — you keep enough single requests in flight to keep the batch full.

## The sizing rule

**Per replica, keep `in-flight ≈ 1.0–1.5 × max_num_seqs`.**

- Below 1.0× the GPU starves between completions.
- Far above it, requests sit in vLLM's internal queue, inflating p99 latency and risking client timeouts.

Enforce with an async semaphore per replica, or per LB pool at `Σ max_num_seqs × 1.25`.

## Per-model sizing

| Model (config) | max_num_seqs | Client in-flight per replica | Note |
|---|---|---|---|
| [[GLM-OCR]] (24 GB / 1g.10gb) | 32 / 64 | 40 / 80 | region requests are short — high turnover, feed aggressively |
| GLM-OCR (2g.20gb) | 128 | ~160 | |
| [[PaddleOCR-VL]] (min / 2g.20gb) | 32 / 96 | 40 / 120 | also capped by CPU layout throughput upstream |
| [[DeepSeek-OCR]] (min / full H100) | 16 / 256 | 20 / 300 | pages emit 1–4K tokens — longer residency per sequence |
| **[[Baidu Unlimited-OCR]]** (min / full H100) | 8 / 32 | **8 / 32 — never overdrive** | each sequence is a whole document holding a slot for *minutes*; queue at the gateway, not in vLLM |
| [[dots.ocr]] (min / 3g.40gb) | 4 / 24 | 4–5 / 28 | memory balloons — treat as a hard ceiling |
| [[MinerU]] | engine-managed | send documents, let the router meter | |

## Free wins inside the engine

- **Continuous batching** is automatic.
- **Chunked prefill** (modern default) — keep it on. Image-heavy prefills would otherwise stall in-flight decodes, which is exactly this workload's shape.
- **`--no-enable-prefix-caching` everywhere** for OCR — requests share no prefix, so the cache only costs memory. (The opposite holds for RAG generation — see [[KV-Cache Reuse for RAG]].)

## When the frontend, not the GPU, is the limit

vLLM's API server can bottleneck at high RPS with tiny requests — the GLM-OCR region-traffic case. Use **`--api-server-count N`** to run multiple frontend processes over one engine *before* adding GPUs, and give GPU pods 8–16 vCPU. See [[Base64 and Image Transport]].

## The scale signal

Alert on `num_requests_waiting > max_num_seqs` sustained beyond 60 s per replica. That is your "scale out or shed load" trigger — and with KEDA it can literally be the scaler. Liveness should fail on sustained growth in the same metric (a wedged-engine detector); readiness is `/health` after model load, gated by a generous `startupProbe` since CUDA-graph capture takes minutes.

## Related

[[Document Fan-Out and Fan-In]] · [[Parallelism Stack for OCR Serving]] · [[vLLM]] · [[MIG and GPU Sharing]]
