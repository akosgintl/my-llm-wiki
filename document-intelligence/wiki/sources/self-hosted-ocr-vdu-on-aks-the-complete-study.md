---
aliases: ["Self-Hosted OCR VDU on AKS The Complete Study"]
tags: [ocr, architecture, consolidated, hungarian, source]
sources: [ocr-vdu-complete-study.md]
created: 2026-08-27
updated: 2026-08-27
---

# Self-Hosted OCR VDU on AKS The Complete Study

**Source:** ocr-vdu-complete-study.md
**Date ingested:** 2026-08-27
**Type:** consolidated master document
**Position in the corpus:** sixth document (2026-08-24 23:08) — **explicitly a consolidation of sources 1–5 plus new material.** Treat it as the canonical statement; the earlier documents are its working papers.

## Summary

Merges the model landscape, vLLM serving recipes and RunPod API v2, API-call and concurrency strategy, the 200-page latency playbook, rasterization and bottleneck analysis, the deployment and EU-sovereignty map, the Hungarian strategy and fine-tuning plan, layout-stage economics, the glmocr-SDK model-swap architecture, the Qwen family map, downstream LLM roles, and confidence engineering into thirteen sections plus a build order.

## Key Claims (the decision card)

1. **Corpus is 80% Hungarian → [[GLM-OCR]] *the model* is out**; **Qwen3.5-4B behind the [[glmocr SDK]] is the front-runner**, with [[PaddleOCR-VL]] and [[dots.ocr]] as specialist arms.
2. **Serving: [[vLLM]] only**; common practical minimum **24 GB, Ampere+**; H100 via MIG packing per model.
3. Fan-out architecture: single-image requests everywhere except [[Baidu Unlimited-OCR]]; client in-flight ≈ 1.25 × Σ max_num_seqs; per-model LB pools; Unlimited-OCR quarantined.
4. Rasterize at model-native resolution (1024–1600 px long edge), JPEG q90 turbo, permissive tools.
5. Deployment: AKS control plane · EU-sovereign GPU burst · RunPod/Modal for non-sensitive work only.
6. Fine-tune only if the probe demands it: Unsloth LoRA, encoder frozen, 10–30K pairs, **format-native targets**.
7. **Confidence is engineered, not emitted** — layout scores + logprobs + value-in-OCR grounding + Hungarian checksums, calibrated, with τ routing to human review.

## New material not in sources 1–5

- **The 200-page latency playbook.** Latency comes from making the document *one continuous-batching wave*: `total ≈ ceil(200 / Σ max_num_seqs) × wave_time`. Text-layer check first (born-digital pages need no GPU) → parallel rasterization (~1–2 s) → fire all 200 at once → stream fan-in by page index → **hedge stragglers for 20–30%** → send only split-table page spans to Unlimited-OCR.
- **Do not use Unlimited-OCR for lowest latency** — its one-shot design is for fidelity, and a one-sequence model cannot parallelize within a document.
- **An explicit build order**: golden set → born-digital self-labeling → probe bake-off → AKS deployment → adapter shim → confidence stack v1 → fine-tune if needed → scale-out.
- **The list of still-open decisions**: workload volumes, latency class, residency confirmation, day-1 single-model commitment, output contract, failure semantics, gateway language.

## Entities Mentioned

All models from source 1, plus [[glmocr SDK]], [[Qwen Model Family]], [[vLLM]], [[SGLang]], [[RunPod]], [[LiteLLM]], [[Unsloth]], [[PP-DocLayout-V3]], [[EXTRACTCONF]], [[neural-maze production-ocr-course]].

## Concepts Covered

[[Golden Set and Eval Harness]] · [[Pipeline as Platform, Model as Config]] · [[Hungarian OCR and the Double Acute Test]] · [[Document Fan-Out and Fan-In]] · [[Parallelism Stack for OCR Serving]] · [[MIG and GPU Sharing]] · [[Load Balancing Inference Pools]] · [[Rasterization at Model-Native Resolution]] · [[Base64 and Image Transport]] · [[EU Sovereign GPU Hosting]] · [[LoRA Fine-Tuning for OCR]] · [[Layout Stage Economics]] · [[Confidence Engineering]] · [[The OCR-to-Text Boundary Limit]] · [[Benchmark Saturation]]

## Caveats stated by the source

"The golden set is the only leaderboard that counts." Throughput and latency figures — including the 1–2-minute 200-page wave — are engineering estimates, explicitly not validated, and must not be turned into an SLA without benchmarking.
