---
aliases: ["RunPod"]
tags: [gpu-cloud, serving, infrastructure]
sources: [ocr-serving-recipes-runpod-v2.md, ocr-api-call-strategy.md, ocr-vdu-complete-study.md, ocr-language-models-deployment-critical-pass-2.md]
created: 2026-08-27
updated: 2026-08-27
---

# RunPod

Serverless/rented GPU provider used as the **non-sensitive tier**: bake-offs, synthetic data, load tests. Never customer documents — see [[EU Sovereign GPU Hosting]].

## API v2 (public beta, 2026)

Unifies everything under one base URL:

```
https://api.runpod.io/v2          # Bearer auth
GET  /v2/openapi.json             # full schema — generate your client from this
POST /v2/templates                # one reusable template per model
POST /v2/pods                     # dev/interactive
POST /v2/endpoints                # serverless, scale-to-zero
```

One `POST /templates` per model gives API-managed configs, and the same template drives both Pods and Serverless workers (`isServerless: true`). Field names match the documented template schema (identical in v1 and v2); v1 at `rest.runpod.io/v1` remains a fallback. Regenerate clients from the OpenAPI spec periodically — it is still beta.

## Serverless caveats that bite

1. **Raw `vllm serve` templates suit Pods; Serverless wants the queue protocol.** Either use RunPod's prebuilt vLLM worker (`runpod/worker-v1-vllm`, `MODEL_NAME` + `VLLM_ARGS` via env — works for [[GLM-OCR]], [[dots.ocr]], [[PaddleOCR-VL]]) or wrap your server in a thin `runpod.serverless` handler. [[DeepSeek-OCR]] and [[Baidu Unlimited-OCR]] need custom images anyway because of their logits-processor flags.
2. **Network volume for weights** (`networkVolumeId`) avoids re-downloading 2–7 GB on every cold start; combine with `flashboot`.
3. **Cold starts**: CUDA-graph capture takes 1–3 min for the 3B models. For latency-sensitive KIE keep `workersMin: 1` on the cheap small model and scale-to-zero only the heavy ones.
4. **No MIG.** RunPod exposes no MIG slices — rent the smaller GPU per replica instead of packing one card. See [[MIG and GPU Sharing]].
5. **Base64 over WAN actually costs here.** In-cluster the +33% wire overhead is irrelevant; to a remote RunPod GPU at 100 Mbps it is ~1.4 s per 200-page wave. Prefer uploading pages to blob once and sending tiny `image_url` requests. See [[Base64 and Image Transport]].

## Load balancing

You control no ingress layer, so put [[LiteLLM]] (or client-side least-busy over the replica URL list) in front. For Serverless endpoints, `scalerType: QUEUE_DELAY` does the data-parallel layer for you — drop your own LB for that pool but keep the semaphore, timeout and retry layer regardless.

## Related

[[vLLM]] · [[LiteLLM]] · [[EU Sovereign GPU Hosting]] · [[Load Balancing Inference Pools]]
