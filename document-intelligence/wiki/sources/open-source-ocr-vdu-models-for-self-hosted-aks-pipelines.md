---
aliases: ["Open-Source OCR VDU Models for Self-Hosted AKS Pipelines"]
tags: [ocr, vlm, benchmarks, model-landscape, source]
sources: [Open-Source OCR per Visual Document Understanding Models for Self-Hosted AKS Pipelines - A Comparative Investigation (August 2026).md]
created: 2026-08-27
updated: 2026-08-27
---

# Open-Source OCR VDU Models for Self-Hosted AKS Pipelines

**Source:** Open-Source OCR per Visual Document Understanding Models for Self-Hosted AKS Pipelines - A Comparative Investigation (August 2026).md
**Date ingested:** 2026-08-27
**Type:** research report
**Position in the corpus:** the first document (2026-08-24 19:51) — the model-landscape survey everything else builds on.

## Summary

A comparative investigation of compact open-source document VLMs for an event-driven OCR/VDU pipeline on AKS with NVIDIA GPUs. Establishes the landscape, the benchmark picture, per-model deep dives, and a four-stage workload-to-model mapping. Written before the Hungarian constraint was known, so its recommendations are later revised.

## Key Claims

- **No single model wins everything — route by workload.** [[GLM-OCR]] for single-page invoices/receipts/KIE and high-throughput batch; [[PaddleOCR-VL]] for multilingual and messy documents; [[Baidu Unlimited-OCR]] for long multi-page contracts; [[MinerU]] as the full-stack ingestion platform.
- Document OCR has **converged on compact (0.9B–3B) specialized VLMs** emitting structured Markdown/JSON directly, largely replacing multi-stage detect-then-recognize pipelines.
- **Treat vendor OmniDocBench scores skeptically** — the benchmark is saturated and Latin/CJK-print-biased; independent multi-script and blind-voting evaluations reorder the field substantially.
- **Deployment practicality varies more than accuracy.** GLM-OCR, PaddleOCR-VL and [[dots.ocr]] are cleanly vLLM/SGLang-served on L4/A10; [[DeepSeek-OCR]] and Unlimited-OCR need custom n-gram logits processors and disabled prefix caching, and Unlimited-OCR additionally needs FlashAttention-3 (Hopper) and a bundled dev-build SGLang wheel.
- **Repetition loops are the dominant production failure mode** across the DeepSeek lineage — a documented 9.2% catastrophic cohort on 600 British Library historical newspaper clippings.
- Always add a repetition/hallucination detector; validate on **your** document distribution (20–50 representative pages) rather than trusting OmniDocBench deltas.
- Consider [[PP-OCRv6]] or a rules-based checker as a hallucination cross-check for high-stakes finance/legal/medical fields.

## Entities Mentioned

- [[GLM-OCR]] — best default for single-page KIE, receipts, seals; fastest small model
- [[PaddleOCR-VL]] — most robust on multilingual and in-the-wild documents
- [[Baidu Unlimited-OCR]] — one-shot long multi-page parsing
- [[DeepSeek-OCR]] — pages-per-dollar bulk digitization, with a retry budget
- [[dots.ocr]] — best independent-benchmark accuracy among open models
- [[MinerU]] — batteries-included ingestion platform
- [[HunyuanOCR]], [[olmOCR]], [[LightOnOCR]], [[Granite-Docling]] — notable contenders
- [[Other OCR Contenders]] — Nanonets, Nemotron, Mistral OCR 4, PP-StructureV3
- [[OmniDocBench]], [[OCR Arena]], [[olmOCR-Bench]], [[Multi-Script OCR Benchmarks]] — the evaluation picture
- [[vLLM]], [[SGLang]], [[neural-maze production-ocr-course]] — serving

## Concepts Covered

- [[Benchmark Saturation]] — the central methodological warning
- [[Repetition Loops in VLM OCR]] — the #1 production failure
- [[Optical Compression]] — DeepSeek's thesis
- [[R-SWA Reference Sliding Window Attention]] — Unlimited-OCR's mechanism
- [[Two-Stage vs End-to-End Document Parsing]]
- [[Permissive Licensing Constraints]] — AGPL PyMuPDF, MinerU's license move

## Caveats stated by the source

Vendor-reported scores; throughput figures are best-case peaks with vLLM startup excluded (sustained end-to-end is roughly half); no published T4/L4 pages/s figures for most models; independent evaluation of Unlimited-OCR is thin; the leaderboard shifts monthly — re-validate quarterly.
