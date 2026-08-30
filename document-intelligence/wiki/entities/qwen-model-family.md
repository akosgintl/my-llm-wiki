---
aliases: ["Qwen Model Family"]
tags: [vlm, multilingual, model, serving]
sources: [ocr-language-models-deployment-critical-pass-2.md, ocr-vdu-complete-study.md, ocr-arxiv-github-technical-review.md, Deep Technical Review - Open-Source OCR - Visual Document Understanding Models - Missed Details and Pipeline Implications.md, Local PDF-Markdown and Document Parsing - State of the Art (August 2026).md]
created: 2026-08-27
updated: 2026-08-27
---

# Qwen Model Family

[[Alibaba Qwen Team]]'s generational ladder, and the front-runner for the recognition slot behind the [[glmocr SDK]] once Hungarian disqualified [[GLM-OCR]].

## Generation map

**Verified against model cards and repos on 2026-08-27.** Two rows in the previous version of this table were wrong and are corrected below.

| Generation | Open weights | Image input | Licence | Verdict for document OCR |
|---|---|---|---|---|
| Qwen3-VL (Dec 2025) | 2B/4B/8B/32B dense + MoE | ✅ | Apache-2.0 | The last OCR-*marketed* release: explicit doc parsing, QwenVL-HTML with `data-bbox`, an OCR cookbook, 39-language OCR set |
| **Qwen3.5 (Feb 2026)** | **0.8B/2B/4B/9B/27B**, 35B-A3B, 122B-A10B, 397B-A17B | ✅ `image-text-to-text` | Apache-2.0 | **The only generation with both the right sizes and published OCR numbers.** 201 languages, 262K context (→1.01M), day-1 [[Unsloth]]/[[vLLM]] |
| Qwen3.6 (Apr 2026) | 27B + 35B-A3B **only** | ✅ | Apache-2.0 | Natively multimodal with OCR and hour-scale video — but **nothing under 27B**. Off the ladder on size, not on capability |
| Qwen3.7 (May 2026) | ❌ **none** | (API only) | proprietary | **A skipped generation** — Plus and Max shipped API-only and never got weights |
| Qwen3.8 (Aug 2026) | 27B + 2.4T-A95B | ✅ `image-text-to-text` | Apache-2.0 | Best document numbers in the family (OmniDocBench 1.5 = **91.1**), still nothing under 27B |

**Correction on the record:** from Qwen3.5 onward there is no "-VL" suffix — and that is **not** a loss of capability. The naming convention inverted because document and vision competence moved into the base model. `Qwen/Qwen3.5-4B` is tagged `image-text-to-text` and sees images. The table below is the evidence.

## The head-to-head that settles it

From Qwen's own Qwen3.5-9B card — **one table, one harness**, so the columns are internally comparable. Vendor-reported, so read it with [[Benchmark Saturation]] in mind; the *relative* ordering is the useful part.

| Benchmark | Qwen3-VL-30B | **Qwen3.5-9B** | **Qwen3.5-4B** |
|---|---|---|---|
| OmniDocBench 1.5 | 86.8 | **87.7** | 86.2 |
| OCRBench | 83.9 | **89.2** | 85.0 |
| CC-OCR | 77.8 | **79.3** | 76.7 |
| CharXiv (RQ) | 56.6 | **73.0** | 70.8 |
| MMLongBench-Doc | 47.4 | **57.7** | 54.2 |
| AI2D | 86.9 | **90.2** | 89.6 |

Two things follow:

1. **Qwen3.5-4B matches Qwen3-VL-30B on document parsing** at a fraction of the parameters, and 9B beats it on every row. Dropping "-VL" cost nothing measurable on documents.
2. **CharXiv (RQ) jumps +16.4 points.** That is the exact benchmark [[The OCR-to-Text Boundary Limit]] is built on. A 4B model scoring 70.8 on chart reasoning changes the [[ColPali]] arithmetic — some of what dies at the seam can be recovered by one end-to-end model rather than a second retrieval stack.

**Do not compare these against the OmniDocBench figures in the Qwen3-VL paper.** That paper reports an edit-distance variant (lower is better); OmniDocBench 1.5 is an overall score (higher is better). Same name, different metric.

## What the newer generations do *not* give you

- **The size ladder closes after 3.5.** 0.8B–9B open weights exist **only** in Qwen3.5. From 3.6 onward the smallest open checkpoint is 27B dense. Qwen3.5 is therefore not a waypoint toward 3.6/3.8 — for a per-page self-hosted pipeline it is the **terminus**. Treat 3.8-27B as a batch-tier option, not as a successor.
- **The document-parsing contract is undocumented.** Qwen3-VL ships an OCR cookbook, QwenVL-HTML output and `data-bbox` attributes on a 0–999 normalized grid. **None of that is documented in the Qwen3.5, 3.6 or 3.8 repos.** The capability may be present and merely unwritten, but two of the four fields [[The OCR-to-RAG Seam]] requires — `bbox` and `page_index` — now rest on an unverified assumption. **Measure it before designing around it.**
- **No per-language OCR breakdown.** Qwen3-VL published one (Figure 2). Nothing in the 3.5+ line has an equivalent.

## The language numbers, precisely

- **Qwen3-VL**: 30M in-house multilingual OCR samples across **39 languages**, >70% accuracy on **32 of 39**. **Figure 2 was read on 2026-08-27** — the full roster and what it settles live in [[Hungarian OCR and the Double Acute Test]]. Short version: the figure plots only the 32 languages that clear 70%, **Hungarian is not among them**, the 7 below the threshold are never named anywhere in the paper, and the string "Hungarian" does not occur in the PDF at all. So the best available case for Hungarian is *supported but under 70%* — a level the paper itself calls below practical usability.
- **Qwen3.5's "201 languages/dialects"** (up from Qwen3's 119) is a **general text claim, not OCR-specific**, and no per-language OCR evidence has been published for the generation. The Qwen3.5-Omni Fleurs top-59 does list Hungarian — but that is *speech*. Hungarian OCR accuracy remains an empirical question for the probe set.

## Evidence that the adaptation path works

There is **no official `Qwen/Qwen3.5-OCR` checkpoint**. There are, however, a dozen-plus community OCR fine-tunes on Hugging Face built on Qwen3.5-0.8B / 2B / 9B / 27B — English document OCR, Arabic, Japanese, LaTeX, handwriting. That is direct evidence the models are **fine-tunable for this task with existing tooling** ([[LoRA Fine-Tuning for OCR]], [[Unsloth]], [[LLaMA-Factory]]) — the exact capability [[PaddleOCR-VL]] lacks.

## Independent evidence, and the contradiction it creates

A practitioner report running Qwen3.5 for document OCR (Martin Alderson, 2026) finds **9B is the sweet spot**, and that **0.8B–4B "struggle to keep on task and end up summarising documents instead"** of transcribing them. That is the instruction-side twin of [[Linguistic Crutch and Faithfulness]], and it is a measured failure at exactly the size this page previously recommended as default. Throughput anchors from the same report: **~3 s/page for 9B** on a Radeon 9070XT, and **~$0.12 per 1,000 pages** through a hosted provider — an API figure, useful only as a crossover reference for [[Cost per Page Model]].

## Serving implications

- **Visual token budget is the real cost knob**, replacing `max-model-len`: ~32× spatial compression, per-image tokens steerable via the processor pixel budget (256–1280 tokens), with dimensions rounding to **multiples of 32** (28 for Qwen2.5-VL). For region crops, cap the budget low.
- Architecture relevant to fine-tuning: SigLIP-2 vision encoder + 2-layer MLP merger compressing 2×2 features to 1 token, **DeepStack** multi-level feature injection, Interleaved-MRoPE; grounding uses a normalized [0,1000] coordinate system. Training Stage-0 updates **only the MLP merger** with encoder and LLM frozen — corroborating the freeze-the-encoder plan and making the merger a legitimate extra LoRA target.
- **Qwen3.5 uses hybrid Gated DeltaNet + Gated Attention** in an 8×(3×DeltaNet→FFN→1×Attention→FFN) pattern, i.e. ~75% linear-attention layers. This **invalidates standard KV-cache assumptions** — MIG slice memory profiles must be re-validated and the vLLM version pinned (vLLM 0.17+ added GDN kernels and a hybrid KV-cache manager; may need `--trust-remote-code`). The 3.6 and 3.8 generations carry this architecture forward, so moving up the ladder does not retire the risk. See [[MIG and GPU Sharing]].
- Document parsing emits "QwenVL HTML" with bbox metadata — **verified for Qwen3-VL, unverified for 3.5+** (see above). The dialect an adapter shim normalizes away.

## The recognition-slot ladder

Revised 2026-08-27 on the benchmark table and the practitioner report.

1. **Qwen3.5-9B (default)** — ~18 GB BF16 (tight on 24 GB; FP8 halves it), 3g.40gb MIG. Promoted over 4B because the 4B's drift into summarisation is *measured*, and because 9B beats Qwen3-VL-30B on every document row.
2. **Qwen3.5-4B (throughput arm)** — ~9 GB BF16, comfortable on 24 GB, 2g.20gb MIG. Keep it as the cheap tier and as the reference for what "fast" costs, but gate it behind a faithfulness check, not only a CER check.
3. **Qwen3.8-27B (batch tier, conditional)** — only if the latency class is batch or overnight. Best document numbers in the family, but 27B dense resets the cost model. Replaces the former Qwen3.6-35B-A3B entry.

**Removed from the ladder:** Qwen3.6 (nothing under 27B) and Qwen3.7 (never released as weights).

Price it consciously: 4B is ~3–5× slower per region than the 0.9B specialists, and 9B more again. Language coverage is not free.

## Related

[[glmocr SDK]] · [[Qwen3-Embedding]] · [[Pipeline as Platform, Model as Config]] · [[Multi-Token Prediction and Speculative Decoding]] · [[Hungarian Model Decision Matrix]] · [[The OCR-to-RAG Seam]] · [[The OCR-to-Text Boundary Limit]]
