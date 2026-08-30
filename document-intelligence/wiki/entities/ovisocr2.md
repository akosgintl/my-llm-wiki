---
aliases: ["OvisOCR2"]
tags: [ocr, vlm, document-parsing, model, benchmarks]
sources: [Local PDF-Markdown and Document Parsing - State of the Art (August 2026).md]
created: 2026-08-27
updated: 2026-08-27
---

# OvisOCR2

An end-to-end document parsing model (arXiv 2607.13639, July 2026) that converts page images directly to Markdown — text, formulas, tables and visual elements. **Found by the coverage audit on 2026-08-27; it was absent from this wiki entirely.** See [[Coverage and Exclusion Register]].

## Why it matters immediately

Read from the `ATH-MaaS/OvisOCR2` model card on 2026-08-27:

| Property | Value |
|---|---|
| Licence | **Apache-2.0** ✅ (`SPDX-License-Identifier: Apache-2.0`) |
| Parameters | **0.8B** (a 4B branch is used during training only) |
| Base model | **Qwen3.5-0.8B** |
| OmniDocBench v1.6 | **96.58 overall** |
| PureDocBench | 75.06 Avg3 |
| Declared languages | **none listed** |

Three of those cells are individually decision-relevant:

**1. 96.58 would be the highest OmniDocBench v1.6 score anywhere in this wiki — but it is self-reported.** Checked against the official `opendatalab/OmniDocBench` leaderboard on 2026-08-27: **OvisOCR2 does not appear on it.** The published `v1.6_full` table is led by [[PaddleOCR-VL]]-1.6 at **96.34**, then MinerU2.5-Pro 95.75 and [[GLM-OCR]] 95.22. So the honest statement is: *a self-report of 96.58 sits 0.24 above a verified leaderboard entry of 96.34* — which is inside the 1–2 point band [[OmniDocBench]] itself calls noise, and it is a self-report against a leaderboard result. **Treat it as a claim to test, not a rank.** If it holds at 0.8B against PaddleOCR-VL's 0.9B, it is genuinely notable; if it does not, this row disappears.

**2. The backbone is Qwen3.5-0.8B.** That is the same family as the recommended recognition arm and as the top [[ColPali]] checkpoint — a third instance of the convergence noted in [[Qwen Model Family]]. It also means the model inherits Qwen3.5's hybrid Gated DeltaNet attention, so the vLLM version-pinning and KV-cache caveats apply here too.

**3. Apache-2.0 at 0.8B** puts it in the same size and licence class as [[GLM-OCR]] and [[PaddleOCR-VL]] — the cheap tier — while claiming quality above the long-context specialists.

## What is not known, and must not be assumed

- **No language roster.** The card declares none, and the paper does not mention Hungarian or Central European coverage. Its evidential status on Hungarian is therefore **identical to [[Baidu Unlimited-OCR]]'s**: a strong benchmark number with no language evidence behind it. See [[Benchmark Saturation]].
- **No independent multi-script evaluation.** It does not appear in socOCRbench, GlotOCR, MORE or MDPBench as recorded in [[Multi-Script OCR Benchmarks]]. OmniDocBench is **Chinese and English only** — so 96.58 says nothing about ő and ű.
- **The Qwen3.5-0.8B base is the size the practitioner evidence warns about.** The report cited in [[Qwen Model Family]] finds Qwen3.5 at 0.8B–4B drifts into *summarising* documents instead of transcribing them. OvisOCR2 is task-trained rather than prompted, which plausibly fixes exactly that — but "plausibly" is a hypothesis, and it is the first thing a probe should check. See [[Linguistic Crutch and Faithfulness]].
- **The card's date field reads 2024**, which contradicts the arXiv identifier (2607 = July 2026). Treated here as a card artifact; the arXiv id is the reliable date.
- **No throughput figure, no VRAM figure, no fine-tuning documentation** located as of the audit.

## Where it sits in the probe order

It is the **highest-variance candidate on the table**: the best claimed quality, the smallest size, a permissive licence — and the thinnest evidence base of any row. That combination argues for probing it *early and cheaply*, because a single golden-set run either promotes it above everything else or eliminates it, and both outcomes are worth a lot.

Do not promote it on the OmniDocBench number alone. That is the exact mistake [[Benchmark Saturation]] documents, and this model is currently its most tempting instance.

## Related

[[Hungarian Model Decision Matrix]] · [[Coverage and Exclusion Register]] · [[Qwen Model Family]] · [[Benchmark Saturation]] · [[OmniDocBench]] · [[Baidu Unlimited-OCR]]
