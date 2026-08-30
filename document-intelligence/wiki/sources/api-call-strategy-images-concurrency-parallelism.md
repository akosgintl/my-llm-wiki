---
aliases: ["API Call Strategy Images Concurrency and Parallelism"]
tags: [serving, concurrency, load-balancing, architecture, source]
sources: [ocr-api-call-strategy.md]
created: 2026-08-27
updated: 2026-08-27
---

# API Call Strategy Images Concurrency and Parallelism

**Source:** ocr-api-call-strategy.md
**Date ingested:** 2026-08-27
**Type:** technical reference
**Position in the corpus:** third document (2026-08-24 21:09) — companion to the serving recipes, covering the client and cluster side.

## Summary

Four topics: images-per-request limits per model, client concurrency sizing against `max_num_seqs`, the full six-layer parallelism stack from request to silicon, and load balancing in front of MIG slices.

## Key Claims

- **Everything except [[Baidu Unlimited-OCR]] is a single-image-per-request system.** The gateway's job is fan-out/fan-in, which is also the ideal shape for vLLM continuous batching.
- **For Unlimited-OCR, budget tokens, not pages**: `pages × 800 output + pages × ~300 vision tokens` against a 32K ceiling → chunk at ~25–30 pages with 1-page overlap.
- **Sizing rule: keep `in-flight ≈ 1.0–1.5 × max_num_seqs` per replica.** Below 1.0× the GPU starves; far above it, p99 inflates. Exception: Unlimited-OCR at *exactly* max_num_seqs — each sequence holds a slot for minutes.
- **The middle of the parallelism stack is a trap.** Tensor and pipeline parallelism are explicitly wrong here — every model fits one device, so TP overhead would make them *slower*. Leverage is at fan-out (L1), continuous batching (L3) and replicas (L5).
- **External data parallelism beats vLLM internal DP** for an event-driven Kubernetes setup, because you keep per-replica health and scaling granularity.
- **MIG > MPS > time-slicing.** MPS has no memory protection — one OOM can take down neighbors. Never time-slice for inference.
- **A plain Kubernetes Service is the wrong load balancer**: it balances TCP connections, and keep-alive pins a client to one slice — one MIG slice at 100%, six idle.
- **Since OCR requests share no prefix, "least-loaded" is provably the optimal routing policy** — so Gateway API Inference Extension's prefix-aware feature buys nothing here. Skip it.
- **Never mix Unlimited-OCR into a page-model pool.** Separate pool, separate queue.
- Readiness ≠ liveness for vLLM: liveness should also fail on sustained `num_requests_waiting` growth, which doubles as the KEDA scale signal.

## Entities Mentioned

- [[vLLM]] — continuous batching, `--api-server-count`, `/metrics`
- [[LiteLLM]] — the pragmatic Tier-2 load balancer
- [[Baidu Unlimited-OCR]] — the multi-image exception, quarantined in its own pool
- [[GLM-OCR]], [[PaddleOCR-VL]], [[DeepSeek-OCR]], [[dots.ocr]], [[MinerU]] — per-model request shapes
- [[RunPod]] — no MIG, no ingress control

## Concepts Covered

- [[Document Fan-Out and Fan-In]] — the gateway pattern and retry loop
- [[vLLM Continuous Batching and Concurrency Sizing]] — the 1.25× rule
- [[Parallelism Stack for OCR Serving]] — the six layers
- [[MIG and GPU Sharing]] — the three mechanisms
- [[Load Balancing Inference Pools]] — the four-tier LB ranking

## Caveats stated by the source

Per-pool rules matter more than the load-balancer choice. Least-outstanding-requests is a good proxy for vLLM load only when request costs are homogeneous — which is false the moment page and long-document traffic share a pool.
