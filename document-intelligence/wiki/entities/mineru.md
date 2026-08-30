---
aliases: ["MinerU"]
tags: [ocr, vlm, document-parsing, idp, platform]
sources: [Open-Source OCR per Visual Document Understanding Models for Self-Hosted AKS Pipelines - A Comparative Investigation (August 2026).md, ocr-serving-recipes-runpod-v2.md, ocr-vdu-complete-study.md, ocr-arxiv-github-technical-review.md, Deep Technical Review - Open-Source OCR - Visual Document Understanding Models - Missed Details and Pipeline Implications.md, Local PDF-Markdown and Document Parsing - State of the Art (August 2026).md]
created: 2026-08-27
updated: 2026-08-27
---

# MinerU

[[OpenDataLab]]'s document ingestion **platform**, not merely a model (arXiv 2509.22186; Pro arXiv 2604.04771). Licensing moved from AGPLv3 to a custom Apache-2.0-based "MinerU Open Source License" in April 2026, removing enterprise copyleft friction — though attribution conditions and high-usage commercial thresholds remain.

## The 1.2B model

NaViT-675M vision encoder + Qwen2-0.5B decoder. Its two stages live **inside one model**, unlike [[PaddleOCR-VL]] and [[GLM-OCR]] which use a separate detector:

1. layout analysis on a **downsampled** global image,
2. recognition on **native-resolution crops** guided by that layout.

This coarse-to-fine trick dodges the O(N²) visual-token blowup of end-to-end native-resolution models — and means **MinerU needs no [[Layout Stage Economics]] decision at all**.

## Full content integrity

Headers, footers and page numbers are *preserved and labeled* under a standardized schema (lists, references, code blocks), where most competitors discard them. That makes MinerU the natural archival and compliance arm, since page furniture can be evidentiary in legal documents.

## Backends

| Backend | VRAM | Notes |
|---|---|---|
| `pipeline` | 4 GB, CPU-capable | rule + light models, PP-OCRv6; ~86.2 OmniDocBench v1.5 |
| `vlm-engine` | 8 GB+ (Volta+) | vLLM / SGLang / LMDeploy / MLX; 90+ |
| `hybrid` | — | native-text + VLM, low hallucination |
| `http-client` | 2 GB | remote VLM |

Plus `mineru-router` for multi-GPU load-balancing, an async task API, streaming disk writes, an MCP server, and LangChain/Dify/FastGPT integrations. ~2.12 pages/s and ~2337 tokens/s on A100 via the vLLM async engine; MLX gives a 100–200% speedup on Apple Silicon.

## MinerU2.5-Pro: the data-engineering argument

Pro reaches **95.69** on OmniDocBench v1.6 (from 92.98) with an **identical 1.2B architecture**, initialized from MinerU2.5 Stage-0. The gain came purely from the data engine (DDAS + CMCV + Judge-and-Refine, <10M→65.5M pages), including +5.54 Table TEDS. This is the strongest published evidence that a data plan can outperform architecture chasing — directly relevant to any 10–30K-pair Hungarian fine-tune. See [[LoRA Fine-Tuning for OCR]].

## Dynamic repetition suppression

Per-layout-element frequency and presence penalties applied at decode in stage 2, tuned so they avoid killing legitimately repetitive structures such as tables and equations. A third repetition-control pattern alongside n-gram processors and [[R-SWA Reference Sliding Window Attention]] — worth porting conceptually into your own vLLM sampling parameters. See [[Repetition Loops in VLM OCR]].

## Weakness

Near-bottom on independent multi-script evaluation (socOCRbench ~0.165) and it **collapses on non-Latin scripts** (MORE 48.85, last by a wide margin against HunyuanOCR's 92.42). Strong on Latin/CJK print only. See [[Multi-Script OCR Benchmarks]].

## Watch item: MinerU-Diffusion

Decodes blocks of 32 masked tokens, iteratively unmasking by **per-position confidence** — a diffusion decoder that emits confidence natively as a byproduct, which is exactly what every autoregressive candidate lacks. Requires the custom `nano_dvlm` engine and is incompatible with [[vLLM]]. A quarterly re-check for the [[Confidence Engineering]] stack, not a plan.

## Related

[[Docling]] · [[PaddleOCR-VL]] · [[OpenDataLab]] · [[Two-Stage vs End-to-End Document Parsing]]
