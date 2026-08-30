---
aliases: ["Parallelism Stack for OCR Serving"]
tags: [serving, architecture, performance, optimization]
sources: [ocr-api-call-strategy.md, ocr-vdu-complete-study.md]
created: 2026-08-27
updated: 2026-08-27
---

# Parallelism Stack for OCR Serving

Six layers from request to silicon. **For 0.9–3B OCR models the leverage is at the top and the bottom; the middle is a trap.**

## L1 — Document/page fan-out (your gateway)

Queue per model *and* per priority; workers pull, rasterize on CPU pods, fan out pages or regions, fan in ordered results. Enqueue **documents**, fan out pages in-process — per-page queue messages cost 20–100 ms each. See [[Document Fan-Out and Fan-In]].

## L2 — Request concurrency

Async semaphores at `1.25 × Σ max_num_seqs`. Plus `--api-server-count N` when small-request RPS saturates the vLLM frontend. See [[vLLM Continuous Batching and Concurrency Sizing]].

## L3 — Continuous batching + chunked prefill

Inside [[vLLM]], and **free**. Keep chunked prefill on; keep prefix caching off.

## L4 — Speculative decoding

**Only [[GLM-OCR]] and the Qwen3.5/3.6 lineage.** Do not bolt draft-model speculation onto the others — OCR output is already cheap per token on MoE decoders. See [[Multi-Token Prediction and Speculative Decoding]].

## L5 — Data parallelism = replicas

The workhorse. Two flavors:

- **External DP (recommended)** — N independent vLLM processes (one per MIG slice or GPU) behind a load balancer. Simple, failure-isolated, autoscales per pool.
- **vLLM internal DP** (`--data-parallel-size N`) — one launch command spreading engine replicas across GPUs with a built-in coordinator. Convenient on a multi-GPU node, but you lose per-replica health and scaling granularity. For an event-driven Kubernetes setup, **external DP wins**.

**Tensor parallelism (`--tensor-parallel-size`): don't.** TP pays off when a model does not fit one device; every model here fits in 10–40 GB. TP overhead would make these models *slower*. Same for pipeline parallelism.

## L6 — GPU sharing

MIG > MPS > time-slicing, with hard rules. See [[MIG and GPU Sharing]].

## The decision card

1. Single image per request everywhere except [[Baidu Unlimited-OCR]].
2. Client in-flight = 1.25 × Σ max_num_seqs per pool; async semaphore; jittered retries; loop-detector-as-retry.
3. Parallelism = fan-out (L1) + continuous batching (L3) + replicas (L5) + MIG (L6). **Skip TP/PP entirely.** MTP only on GLM-OCR. MPS only for 0.9B models on non-MIG cards. Never time-slice.
4. Load balancer = Gateway API Inference Extension on AKS, [[LiteLLM]] as the pragmatic alternative; per-model pools; Unlimited-OCR quarantined. See [[Load Balancing Inference Pools]].

## Related

[[Document Fan-Out and Fan-In]] · [[MIG and GPU Sharing]] · [[Load Balancing Inference Pools]] · [[vLLM]]
