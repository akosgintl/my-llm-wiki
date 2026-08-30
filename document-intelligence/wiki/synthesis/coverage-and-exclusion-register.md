---
aliases: ["Coverage and Exclusion Register"]
tags: [synthesis, methodology, decision, audit, licensing]
sources: [Open-Source OCR per Visual Document Understanding Models for Self-Hosted AKS Pipelines - A Comparative Investigation (August 2026).md, ocr-vdu-complete-study.md, Local PDF-Markdown and Document Parsing - State of the Art (August 2026).md, ocr-arxiv-github-technical-review.md, The Everything Else Map of SOTA RAG Optimization (2024–2026) - A Decision-Ready Survey for a Greenfield Agentic RAG System.md]
created: 2026-08-27
updated: 2026-08-27
---

# Coverage and Exclusion Register

**Every model this wiki has ever named, with an explicit verdict and the evidence behind it.**

This page exists because the [[Hungarian Model Decision Matrix]] kept turning out to be missing rows, and each absence looked like an oversight rather than a decision. A model that is not on the matrix must be *excluded on stated evidence*, not merely absent. Silence is not a verdict.

**Rule for this page: no row without a source and a date.** Where the evidence is thin, the row says so instead of guessing.

**Audited 2026-08-27** against model cards, repository READMEs and papers. Every "checked" claim below was read on that date.

## On the matrix — candidates for the recognition slot

Twelve rows, each with its own verdict on the [[Hungarian Model Decision Matrix]]: [[PaddleOCR-VL]], [[dots.ocr]], [[Qwen Model Family]] (Qwen3-VL and Qwen3.5), [[HunyuanOCR]], [[DeepSeek-OCR]] v1 and OCR-2, [[GLM-OCR]], [[LightOnOCR]], [[MinerU]], [[olmOCR]], [[Baidu Unlimited-OCR]], and **[[OvisOCR2]]** (added by this audit).

## Excluded, with evidence

### Licence-blocked

| Model | Verdict | Evidence (read 2026-08-27) |
|---|---|---|
| [[Marker and Chandra]] | ❌ **Weights are not permissive.** Code is Apache-2.0, weights are not | `datalab-to/marker` README: *"Our model weights use a modified AI Pubs Open Rail-M license (free for research, personal use, and startups under $5M funding/revenue)."* The `datalab-to/chandra` HF card tags `openrail`, 9B, olmOCR-Bench 83.1, "40+ languages" with Hungarian not listed |
| [[HunyuanOCR]] | ⚠️ **On the matrix, but legally gated** — kept because it is the faithfulness outlier | NOASSERTION licence; a counsel question, not a technical one. See [[Permissive Licensing Constraints]] |
| NVIDIA **Nemotron Parse 2.0** / `nemotron-ocr-v2` | ❌ **Not a standard permissive licence, and the multilingual work points elsewhere** | Governed by **OpenMDW-1.1 plus the NVIDIA Open Model License Agreement** (tokenizer CC-BY-4.0). The v2.0 vocabulary expansion (52,329 → 72,256 tokens) is evaluated on **IndicVisionBench** (10 Indic languages) and MOSCAR; **Hungarian is not named** |

### Not open weights — the strong numbers belong to hosted products

| Model | Verdict | Evidence (read 2026-08-27) |
|---|---|---|
| **Nanonets OCR-3 / OCR2-Plus** | ❌ **The leaderboard numbers are not downloadable.** `nanonets/` publishes only OCR-s (Jun 2025), OCR2-3B (Oct 2025) and OCR2-1.5B-exp (Dec 2025) — **there is no OCR-3 open checkpoint**, and OCR2-Plus is served through the Docstrange API | The open `nanonets/Nanonets-OCR2-3B` card (base Qwen2.5-VL-3B): **olmOCR-bench 69.5**, MDPBench 64.2, and 11 named languages — English, Chinese, French, Spanish, Portuguese, German, Italian, Russian, Japanese, Korean, Arabic — **Hungarian absent**. The 87.4 and 82.0 figures on [[olmOCR-Bench]] are the hosted products |
| **Mistral OCR 4** | ❌ Managed API, not self-hostable. Retained as a **comparison point** because it exposes per-word confidence, which no self-hosted VLM does | See [[Other OCR Contenders]] and [[Confidence Engineering]] |
| **Gemini 3 / 3.1 Pro, Flash-Lite** | ❌ Proprietary API. Retained as the **ceiling reference** — it still leads independent multi-script evaluation by a wide margin | [[Multi-Script OCR Benchmarks]]: socOCRbench 0.6357 for Gemini 3.1 Pro against 0.478 for the best open model |

### Language-blocked

| Model | Verdict | Evidence (read 2026-08-27) |
|---|---|---|
| [[Granite-Docling]] | ❌ **English-first by declaration.** Also self-declared as a *component*, not a page parser: *"Granite-Docling is **not** intended for general image understanding"* | HF card: Apache-2.0, 258M, primary language **English**, with *"Japanese, Arabic and Chinese support (experimental)"*. **No Hungarian, no European language beyond English.** OCRBench 500, full-page OCR F1 0.84 |
| [[LightOnOCR]] | ❌ Already on the matrix as a **verified negative** | Card declares 11 languages (`en fr de es it nl pt sv da zh ja`) — no CEE language |
| [[GLM-OCR]] | ❌ Already on the matrix as a verified negative | Paper footnote 2: exactly 8 core languages; breaks on Polish |

### Different slot — not competing for recognition

| Tool | Role it actually fills | Why that is the right place for it |
|---|---|---|
| [[Tesseract]] | **Tier-1/2 validator and the only calibrated confidence source** | It is the one tool in this wiki with a *positive* Hungarian statement besides [[PaddleOCR-VL]] — handles ő/ű on clean scans, and emits the per-character confidence VLMs cannot. Set `lang=hun` explicitly. Excluding it from the recognition matrix is a slot decision, not a quality one. See [[Confidence Engineering]] |
| [[PP-OCRv6]] | 34.5M CTC recognizer used as a **hallucination cross-check** | A second opinion on a region, not a candidate to parse the page |
| [[PP-DocLayout-V3]] | Layout detection and reading order | Supplies `bbox` independently of the recognition model — the fallback discussed in [[The OCR-to-RAG Seam]] |
| [[liteparse]], [[pymupdf4llm]], [[pypdfium2]] | Tier-1 born-digital extraction and rasterization | No OCR involved; see [[Tiered Page Routing]] |
| [[Docling]] | Orchestration framework and document schema | The alternative platform to the [[glmocr SDK]], not a model |
| **Qianfan-OCR** | Cited for its Table 6, not as a candidate | That table is the entire evidence base for [[The OCR-to-Text Boundary Limit]] |

### Superseded or unmaintained

| Model | Verdict | Evidence |
|---|---|---|
| **PP-StructureV3** | Superseded in accuracy by [[PaddleOCR-VL]] | OmniDocBench v1.5 overall 86.73 |
| **GOT-OCR2** (2024) | Dated | [[olmOCR-Bench]] 48.3 |
| **Nougat** (Meta, 2023) | Effectively unmaintained; English/arXiv only | Dependency and transformers breakage, open issues |
| **Unstructured** | Orchestration layer, not a parser | Table and formula fidelity trails specialised VLMs |

### Named but genuinely unassessed

**MonkeyOCR-Pro · FireRed-OCR · Youtu-Parsing · InternVL.** The corpus names them and provides no distinguishing evidence. **This row is an admission, not a verdict** — they are neither included nor ruled out, and closing that would take a benchmark read each. Recorded here so the gap is visible rather than silently absent.

## Gaps this audit found in the wiki itself

1. **[[OvisOCR2]] was missing entirely** — Apache-2.0, 0.8B, built on a **Qwen3.5-0.8B** backbone, reporting **OmniDocBench v1.6 = 96.58**. Page created and row added to the matrix, with the number marked as a **self-report**: OvisOCR2 does not appear on the official leaderboard, where the verified leader is [[PaddleOCR-VL]]-1.6 at 96.34.
2. **[[Qwen3-VL-Embedding]] was missing entirely** — a unified multimodal embedder and reranker that is the direct single-vector alternative to [[ColPali]]'s late interaction, and the RAG half had no page for it. Page created.
3. **[[olmOCR-Bench]] mixed open weights with hosted APIs** in one score table, which made the leaderboard read as if the top entries were downloadable. They are not. Fixed by marking availability per row.
4. **[[Baidu Unlimited-OCR]] had no language section at all** and was absent from the matrix despite holding the wiki's second-highest OmniDocBench score. Fixed in the same audit.

## Freshness — later evidence overrides earlier evidence

The wiki was built by ingesting sources in order, so **an older source can leave a claim standing that a newer read contradicts**. The rule is that the later evidence wins and the earlier claim is *corrected*, not left beside it. Four corrections this audit made, each replacing a claim the ingested sources had put on the record:

| Claim as it stood | Corrected to | Source |
|---|---|---|
| [[Marker and Chandra]] weights free under **$2M** revenue/funding | **$5M** funding/revenue | `datalab-to/marker` README, 2026-08-27 |
| Nanonets OCR-3 is a ~3–4B **Apache-2.0** model leading olmOCR-Bench | **No open OCR-3 exists.** The open OCR2-3B scores **69.5**, not 87.4 | `nanonets/` org listing + OCR2-3B card, 2026-08-27 |
| [[Qwen Model Family]]: Qwen3.7 "Max/Plus", Qwen3.8 "lineup unconfirmed", Qwen3.6 "zero OCR evidence" | 3.7 **never shipped weights**; 3.8 is 27B + 2.4T-A95B and multimodal; 3.6 is multimodal with OCR | model cards + repos, 2026-08-27 |
| [[DeepSeek-OCR]] as one row mixing v1 and OCR-2 evidence | Two rows — OCR-2 supersedes v1 on every clean-print number and closes **none** of the multi-script gap | HF card + [[Multi-Script OCR Benchmarks]], 2026-08-27 |

**And one grade-of-evidence rule that came out of the same pass.** A number published on an official leaderboard and a number published in a vendor's own paper are not the same kind of fact. On OmniDocBench v1.6, [[PaddleOCR-VL]]-1.6's **96.34** is a leaderboard entry read from `opendatalab/OmniDocBench`; [[OvisOCR2]]'s **96.58** and [[Baidu Unlimited-OCR]]'s **93.92** are self-reports that appear on **no** official leaderboard. Where a self-report outranks the verified leader by less than the benchmark's own noise band, the correct verdict is *unproven*, not *better*.

## How to keep this page honest

Add a row **whenever a model is named anywhere in the wiki**, even in passing, and give it a verdict with a dated source. The failure mode this page prevents is the one that produced it: a model gets mentioned in a benchmark table, never gets a verdict, and its absence from the decision surface is later mistaken for a decision.

Re-audit when the [[Hungarian Model Decision Matrix]] is next touched, and treat any row older than a quarter as stale — the field ships a new SOTA claim roughly monthly.

## Related

[[Hungarian Model Decision Matrix]] · [[Other OCR Contenders]] · [[Permissive Licensing Constraints]] · [[Benchmark Saturation]] · [[Open Inputs and Corpus Profile]] · [[Multi-Script OCR Benchmarks]]
