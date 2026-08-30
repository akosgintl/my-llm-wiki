---
aliases: ["Serving Recipes for OCR VDU on vLLM SGLang and RunPod"]
tags: [serving, vllm, sglang, infrastructure, source]
sources: [ocr-serving-recipes-runpod-v2.md]
created: 2026-08-27
updated: 2026-08-27
---

# Serving Recipes for OCR VDU on vLLM SGLang and RunPod

**Source:** ocr-serving-recipes-runpod-v2.md
**Date ingested:** 2026-08-27
**Type:** technical reference / runbook
**Position in the corpus:** second document (2026-08-24 20:25) — turns the model landscape into deployable configs.

## Summary

Per-model serving recipes: a minimal-GPU config, an H100-optimized config, and a [[RunPod]] API v2 template payload for each of six models. Plus a GPU sizing matrix and a workload-to-GPU mapping.

## Key Claims

- **A common practical minimum serves everything: 24 GB, Ampere+ (CC ≥ 8.0)** — forced by [[PaddleOCR-VL]]'s hard Ampere floor and [[dots.ocr]]'s batch memory ballooning. RunPod: RTX 4090. Azure/AKS: A10 (no L4 on Azure). Elsewhere: L4.
- **[[GLM-OCR]] needs the glmocr SDK in front for paper-level accuracy** — serving the bare VLM skips layout analysis entirely.
- **PaddleOCR-VL likewise requires the full two-stage pipeline**, driven via `--vl_rec_backend vllm-server --vl_rec_server_url`.
- **The n-gram logits processor is not optional** for [[DeepSeek-OCR]] and [[Baidu Unlimited-OCR]] — it is the repetition guardrail. Prefix caching and the multimodal processor cache must be off.
- **Unlimited-OCR's SGLang path is Hopper-only** (`--attention-backend fa3`) and uses a **dev-build wheel bundled in the repo** — pin it in your image.
- **dots.ocr's local model directory must contain no periods** (`DotsOCR`).
- On H100, a 0.9B model **saturates on preprocessing before it saturates the GPU** — pack 2–4 replicas via MIG rather than running one large instance.
- **RunPod API v2** (public beta) unifies templates, pods and serverless endpoints under `https://api.runpod.io/v2`; one `POST /templates` per model drives both Pods and Serverless workers.
- Serverless caveats: raw `vllm serve` templates suit Pods, not Serverless; use a network volume for weights to avoid re-downloading on cold start; CUDA-graph capture takes 1–3 min.

## Entities Mentioned

- [[vLLM]] — the primary serving engine; universal OCR flags
- [[SGLang]] — secondary, and mandatory for Unlimited-OCR's long-context path
- [[RunPod]] — API v2 templates, serverless caveats, no MIG
- [[GLM-OCR]], [[PaddleOCR-VL]], [[DeepSeek-OCR]], [[Baidu Unlimited-OCR]], [[dots.ocr]], [[MinerU]] — the six recipes
- [[glmocr SDK]] — required in front of GLM-OCR

## Concepts Covered

- [[MIG and GPU Sharing]] — H100 packing strategy per model
- [[Multi-Token Prediction and Speculative Decoding]] — the MTP flags
- [[Repetition Loops in VLM OCR]] — n-gram processor configuration
- [[vLLM Continuous Batching and Concurrency Sizing]]

## Caveats stated by the source

"These stacks move monthly" — verify flags against each repo before production. Exact MinerU backend and router flags shift between minor versions. RunPod API v2 is public beta; regenerate clients from `/v2/openapi.json`.
