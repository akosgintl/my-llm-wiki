---
aliases: ["neural-maze production-ocr-course"]
tags: [reference-architecture, aks, repo]
sources: [ocr-language-models-deployment-critical-pass-2.md, ocr-vdu-complete-study.md, Deep Technical Review - Open-Source OCR - Visual Document Understanding Models - Missed Details and Pipeline Implications.md, Open-Source OCR per Visual Document Understanding Models for Self-Hosted AKS Pipelines - A Comparative Investigation (August 2026).md]
created: 2026-08-27
updated: 2026-08-27
---

# neural-maze production-ocr-course

`github.com/neural-maze/production-ocr-course` — the closest public architectural match to a self-hosted OCR pipeline on AKS, and therefore the highest-value reference repo in the corpus.

## What it actually deploys

`k8s/aks/kustomization.yml`: **five manifests** (Rust/Axum API + worker, [[vLLM]], Redis, KEDA scaler, Service) plus **one shared `configMapGenerator`** — hash-suffixed, so a config change rolls both deployments.

- The worker is the **unmodified [[glmocr SDK]]**, env-pointed at `ocr-vlm-service:8000` — i.e. swap Option A, live.
- vLLM serves `SERVED_NAME=Qwen/Qwen3.5-4B` with `GPU_MEMORY=0.9, MAX_MODEL_LEN=16384, MAX_NUM_BATCHED_TOKENS=262144, MAX_NUM_SEQS=512`.
- Workers micro-batch (`MAX_BATCH_SIZE=4`, `BATCH_WINDOW_MS=100`) and KEDA multiplies replicas on queue depth — **concurrency lives in replica count, not per-worker parallelism**.
- Other elements across its documentation: [[PP-DocLayout-V3]], /dev/shm zero-copy handoff, APIM security, scale-to-zero, and asymmetric node pools (T4 for layout, A100 for inference).

Its Rust/Axum gateway is also the concrete argument in [[Rasterization at Model-Native Resolution]] for putting the systems layer outside Python.

## Copy with eyes open

- **`MAX_NUM_SEQS=512` with 262K batched tokens is big-card tuning.** Scale to ~32–64 on a 24 GB fleet or it OOMs.
- **No adapter layer exists** — the format-drift risk of running Qwen weights behind GLM-dialect prompts is simply unhandled. See [[glmocr SDK]] Option B.
- The manifests carry `managed-by: gemini-cli-agent`: it was **scaffolded by an agent**. Validate, do not venerate.
- It is free course material, not battle-tested production. Its *patterns* (env-var config, KEDA shape, decoupled gateway) are sound; its *numbers* are not yours.

## Related

[[glmocr SDK]] · [[Qwen Model Family]] · [[vLLM]] · [[Document Fan-Out and Fan-In]]
