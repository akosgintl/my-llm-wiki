---
aliases: ["Load Balancing Inference Pools"]
tags: [serving, infrastructure, kubernetes, architecture]
sources: [ocr-api-call-strategy.md, ocr-vdu-complete-study.md]
created: 2026-08-27
updated: 2026-08-27
---

# Load Balancing Inference Pools

## The problem with the obvious answer

**A plain Kubernetes Service (ClusterIP) balances at the TCP-connection level, and HTTP keep-alive pins one client connection to one replica.** Result: one MIG slice at 100% and six idle.

You need L7, per-request, and ideally load-aware balancing.

## Ranked options

**Tier 1 — Kubernetes Gateway API Inference Extension (GIE).** The Kubernetes-native, purpose-built answer, and AKS supports Gateway API via managed gateways. Its endpoint-picker routes each request on **live [[vLLM]] signals** — `num_requests_waiting` and KV-cache utilization scraped from `/metrics`.

That is exactly the right signal here: **since OCR requests share no prefix, "least-loaded replica" is the *optimal* policy**, and GIE's other headline feature — prefix-aware routing — buys you nothing. Skip it. Define one `InferencePool` per model and route on model name.

**Tier 2 — [[LiteLLM]] proxy.** Fastest to stand up, OpenAI-compatible, per-model routing, least-busy strategies, health checks, retries and fallbacks, budgets and rate limits per caller. Ideal while the fleet is under ~20 replicas, and the only real option on [[RunPod]].

**Tier 3 — Envoy/Istio `LEAST_REQUEST` or NGINX `least_conn`.** If you already run a mesh or ingress. Least-outstanding-requests is a good proxy for vLLM load **when request costs are homogeneous** — true within a page-model pool, false the moment you mix page and long-document traffic. Add active health checks against `/health` and outlier ejection so a wedged replica is evicted.

**Tier 4 — vLLM production-stack / llm-d routers.** KV-aware and prefix-aware scheduling. Overkill for OCR; revisit only if you later co-host chat/RAG models on the same fleet — where prefix reuse *does* matter. See [[KV-Cache Reuse for RAG]].

## The topology

```
Service Bus / Redis queues (per model, per priority)
        │  KEDA scales pools on queue depth
        ▼
CPU gateway pods — rasterize at fixed size, fan-out, semaphores, retries, loop-detector
        ▼
Gateway API + Inference Extension (or LiteLLM)
   ├── InferencePool: glm-ocr       → 7 × 1g.10gb MIG pods
   ├── InferencePool: paddleocr-vl  → 3 × 2g.20gb MIG pods
   ├── InferencePool: dots-ocr      → 2 × 3g.40gb MIG pods
   ├── InferencePool: deepseek-ocr  → full-GPU pods (batch, spot-friendly)
   └── InferencePool: unlimited-ocr → full-GPU H100 pods — SEPARATE pool, never mixed
        ▼
fan-in / reassembly → downstream extraction
```

## Rules that outrank the load-balancer choice

**1. Never mix [[Baidu Unlimited-OCR]] into a page-model pool.** A 20-minute document request behind the same balancer as 2-second region requests **wrecks every latency-based balancing signal**. Separate pool, separate queue, `in-flight = max_num_seqs` exactly.

**2. Readiness ≠ liveness for vLLM.** Readiness = `/health` after model load, gated by a generous `startupProbe` (CUDA-graph capture takes minutes). **Liveness should *also* fail on sustained `num_requests_waiting` growth** — a wedged-engine detector.

**3. The same metric is your scaler.** Alert on `num_requests_waiting > max_num_seqs` sustained beyond 60 s per replica; with KEDA that alert *is* the scale signal.

## Related

[[MIG and GPU Sharing]] · [[Parallelism Stack for OCR Serving]] · [[LiteLLM]] · [[Document Fan-Out and Fan-In]]
