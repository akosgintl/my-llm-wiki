---
aliases: ["vLLM"]
tags: [serving, inference, infrastructure, tool]
sources: [ocr-serving-recipes-runpod-v2.md, ocr-api-call-strategy.md, ocr-rasterize-encoding-bottlenecks.md, ocr-vdu-complete-study.md, Deep-Dive - 10 RAG Optimization Techniques for a Self-Hosted Agentic Stack.md, The Everything Else Map of SOTA RAG Optimization (2024–2026) - A Decision-Ready Survey for a Greenfield Agentic RAG System.md]
created: 2026-08-27
updated: 2026-08-27
---

# vLLM

The serving engine the entire architecture standardizes on. Everything — OCR models, embedders, rerankers, the generator — is "plain vLLM in a container behind an OpenAI-compatible API," which is what makes multi-cloud a [[LiteLLM]] route entry rather than a re-architecture. See [[Pipeline as Platform, Model as Config]].

## Universal flags for OCR workloads

- **`--no-enable-prefix-caching`** — OCR requests share no prefix; the cache only costs memory. (The opposite is true for RAG generation, where prefix caching is a free win.)
- **`--mm-processor-cache-gb 0`** for the DeepSeek lineage — every request has a distinct image.
- **Generous `startupProbe`** — CUDA-graph capture takes 1–3 minutes for the 3B models.
- **Big vCPU on GPU pods (8–16, not the K8s default 2)** — the frontend does JSON parsing, base64 decoding and image decoding on CPU.
- **`--api-server-count N`** — run multiple frontend processes over one engine when small-request RPS is high (the [[GLM-OCR]] region traffic case). Use it before adding GPUs.
- Keep **chunked prefill** enabled (modern default): image-heavy prefills would otherwise stall in-flight decodes, which is exactly this workload's shape.

## The frontend is the hidden bottleneck

The API server reads the whole body, parses JSON, base64-decodes and PIL-decodes — all CPU in a Python frontend process. **Watch signal:** GPU utilization sawtoothing below 90% while the gateway is idle means the frontend is starving the engine. See [[Base64 and Image Transport]].

## Model-specific requirements

| Model | Requirement |
|---|---|
| [[GLM-OCR]] | `--speculative-config '{"method":"mtp","num_speculative_tokens":3}'`; transformers ≥5.3 |
| [[PaddleOCR-VL]] | `--trust-remote-code`; CC ≥ 8.0; low `gpu-memory-utilization` if layout is co-located |
| [[DeepSeek-OCR]] | n-gram logits processor **mandatory**; caches off |
| [[Baidu Unlimited-OCR]] | dedicated image `vllm/vllm-openai:unlimited-ocr` |
| [[dots.ocr]] | `--trust-remote-code`; `max_num_seqs` as a hard ceiling (4 on 24 GB) |
| [[Qwen3-Embedding]] | `--runner pooling` (newer) or `--task embed`; vllm ≥0.8.5, transformers ≥4.51 |
| Qwen3.5 | vLLM 0.17+ for Gated DeltaNet kernels + hybrid KV-cache manager |

## Continuous batching is the batcher

Never build image batches client-side. Keep `in-flight ≈ 1.0–1.5 × max_num_seqs` per replica via an async semaphore. See [[vLLM Continuous Batching and Concurrency Sizing]].

## Observability

Expose `/metrics` to Prometheus. Readiness = `/health` after model load; **liveness should also fail on sustained `num_requests_waiting` growth** (a wedged-engine detector). Alert on `num_requests_waiting > max_num_seqs` sustained beyond 60 s per replica — with KEDA that alert can literally be the scaler.

## On the RAG side

[[LMCache]] plugs in via the KV-connector API to make retrieved-chunk prefill reusable — the biggest RAG-specific serving win. See [[KV-Cache Reuse for RAG]].

## Related

[[SGLang]] · [[LMCache]] · [[MIG and GPU Sharing]] · [[Parallelism Stack for OCR Serving]] · [[RunPod]]
