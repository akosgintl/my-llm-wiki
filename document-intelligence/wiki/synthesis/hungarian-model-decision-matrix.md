---
aliases: ["Hungarian Model Decision Matrix"]
tags: [synthesis, hungarian, ocr, vlm, benchmarks, licensing, decision]
sources: [Open-Source OCR per Visual Document Understanding Models for Self-Hosted AKS Pipelines - A Comparative Investigation (August 2026).md, ocr-language-models-deployment-critical-pass-2.md, ocr-vdu-complete-study.md, ocr-arxiv-github-technical-review.md, Deep Technical Review - Open-Source OCR - Visual Document Understanding Models - Missed Details and Pipeline Implications.md, tiered-pdf-pipeline-architecture.md]
created: 2026-08-27
updated: 2026-08-27
---

# Hungarian Model Decision Matrix

The one table the corpus never puts in one place. Every recognition-slot candidate scored on the five axes that actually gate the decision: **does it declare Hungarian, can you fine-tune it, is the licence clean, what does it cost to run, and what is the independent evidence.**

This is a synthesis page — every cell is sourced from an entity page. Where a cell says *unverified*, that is a real open item, not a gap in this table. See [[Hungarian OCR and the Double Acute Test]] for why Hungarian is the axis that re-ranked the whole field.

## The matrix

| Model | **Version** | Params | Licence | Hungarian declared? | Fine-tunable? | Speed | Independent evidence |
|---|---|---|---|---|---|---|---|
| [[PaddleOCR-VL]] | **1.6** (Aug 2026) — OmniDocBench figures below are 1.5 | 0.9B | Apache-2.0 ✅ | **✅ Verified — membership only.** Re-read from arXiv 2510.14528 Table A1 on 2026-08-27: Hungarian is named in the 47-language Latin group (a text table, not an image). **But no Hungarian-specific accuracy exists** — Table 6a scores by script *group* (Latin 0.013, best in table) on Baidu's own in-house set | ❌ **No** — the hard blocker | ~1.38 pages/s (vLLM, A100); ~2.0 with FastDeploy | **Leads the official OmniDocBench v1.6 leaderboard at 96.34** (read from the repo 2026-08-27); strong on messy multilingual scans |
| [[dots.ocr]] | **1.5** | 1.7B | MIT ✅ | ⚠️ 100+ claimed, Hungarian not itemised | ✅ | **~0.35 pages/s (A100)** — slowest here | **Best independent scores**; 2nd on GlotOCR mid-resource; socOCRbench 0.478 — the best open model there. OmniDocBench v1.6 leaderboard 90.77 |
| [[Qwen Model Family]] (Qwen3-VL) | Qwen3-VL, **Dec 2025** (arXiv 2511.21631) | 2B–32B | Apache-2.0 ✅ | ❌ **Verified negative (2026-08-27)** — not among the 32 languages above 70%; best case is one of 7 unnamed languages *below* 70% | ✅ | size-dependent | strong general VLM, weaker doc-specialisation |
| **Qwen3.5-9B / 4B** | Qwen3.5, **Feb 2026** — last generation with sub-27B weights | 9B / 4B | Apache-2.0 ✅ | ⚠️ 201 languages — but a **text** claim, not OCR, and no per-language OCR data published | ✅ (community OCR fine-tunes already exist) | ~3 s/page (9B, consumer GPU) | **9B beats Qwen3-VL-30B on every document benchmark**; 4B matches it. Hungarian still unproven |
| [[HunyuanOCR]] | **1.5** | ~1B | ❌ **NOASSERTION** | ⚠️ top on MORE; MORE-149 membership unconfirmed | ✅ | — | **most faithful model measured** (CHAOS-Bench 14.15) |
| [[DeepSeek-OCR]] v1 | **v1**, Oct 2025 (arXiv 2510.18234) | 3B | MIT **disputed** (AGPL dep) | ⚠️ ~100 claimed; **near-bottom multi-script** — below Tesseract v5 on socOCRbench | ✅ **strongest base** | ~200k pages/day (A100-40G) | olmOCR-Bench ~75.4–75.7; 9.2% catastrophic repetition cohort measured on historical scans |
| **[[DeepSeek-OCR]] 2** | **OCR-2**, Jan 2026 (arXiv 2601.20552) | 3B (~570M active MoE) | repo Apache-2.0 ✅, **same AGPL PyMuPDF toolchain wrinkle** ⚠️ | ⚠️ "multilingual", **roster never enumerated**; no Hungarian statement | ✅ **strongest base** — 35 finetunes + 8 adapters published | no v2 throughput figure published | **Split verdict.** English/clean: olmOCR-Bench 76.3, OmniDocBench 73.01, repetition 6.25→4.17% (own measurement). **Multi-script: still near the bottom** — socOCRbench ~0.176 vs [[dots.ocr]] ~0.478, and 40+ points below the GlotOCR leader |
| [[GLM-OCR]] | Feb 2026 (arXiv 2603.10910) — **no version number published** | 0.9B | MIT ✅ | ❌ **No** — 8 core languages, breaks on Polish | ✅ (MTP-merge risk) | **1.86 pages/s**, ~3 GB VRAM — fastest | **Split, not simply weak:** 3rd on the official OmniDocBench v1.6 leaderboard at **95.22**, yet [[OCR Arena]] #24 and olmOCR-Bench ~75.2. A zh/en print benchmark and a blind community ELO disagree about this model |
| [[LightOnOCR]] | **LightOnOCR-2-1B** | 1B | Apache-2.0 ✅ | ❌ **No** — 11 declared, no CEE language | ✅ | **5.71 pages/s (H100)**, <$0.01/1k pages | 83.2 on [[olmOCR-Bench]] (English-only) |
| [[MinerU]] | **2.5** / 2.5-Pro | 1.2B | custom Apache-based ⚠️ | ❌ collapses on non-Latin | — | ~2.12 pages/s (A100) | **2nd on the official OmniDocBench v1.6 leaderboard — 95.75 (MinerU2.5-Pro)**, which the language verdict above does not cancel; full platform, not just a model |
| **[[Baidu Unlimited-OCR]]** | **2026-06-22** (arXiv 2606.23050) | 3.3B | MIT ✅ | ⚠️ **No roster at all** — bare `multilingual` tag, no language enumerated, no multilingual benchmark published. **Frozen encoder inherited from [[DeepSeek-OCR]]**, the weakest multi-script model in the field | ✅ decoder-side (ms-swift supported) — but the encoder ceiling is inherited | whole-document, not pages/s: 40+ pages per call, ~35% faster than DeepSeek-OCR at 6k tokens | **Best OmniDocBench in the wiki** — 93.23 (v1.5) / 93.92 (v1.6) — but **self-reported: it is not on the official OmniDocBench leaderboard**; ParseBench mean 46.17. **Zero independent multi-script evidence** |
| **[[OvisOCR2]]** | **Jul 2026** (arXiv 2607.13639) | **0.8B** (Qwen3.5-0.8B base) | Apache-2.0 ✅ | ⚠️ **No roster** — card declares no languages; paper never mentions Hungarian or CEE coverage | not documented | not published | **Self-reported OmniDocBench v1.6 96.58 — absent from the official leaderboard**, where the verified leader is [[PaddleOCR-VL]]-1.6 at 96.34. PureDocBench 75.06. **No multi-script evaluation at all** |
| [[olmOCR]] | **olmOCR 2** (olmOCR-2-7B) | 7B | Apache-2.0 ✅ | ❌ English linearization | ✅ | ~1.78 pages/s, H100-class | fully open; RLVR recipe is the transferable part |

**Speed figures are not comparable across rows** — they mix A100 and H100, batch sizes and backends. Use them for order-of-magnitude only; see [[Cost per Page Model]].

**Versions are part of the verdict, not decoration.** Every cell in a row describes *that* version, and this field moves fast enough that an undated row is a wrong row — the [[DeepSeek-OCR]] lineage is the proof: v1 scores below Tesseract on socOCRbench while OCR-2 does not, and mixing them produced a wrong verdict here until 2026-08-27. Where a benchmark number was measured on an older point release, the version column says so. Re-verify versions before any probe: this table was checked on **2026-08-27**.

**Prefer the freshest evidence, and prefer leaderboards to self-reports.** Two rules this table now follows explicitly, both learned the hard way on 2026-08-27. *Freshest:* where a later source contradicts an earlier one, the later wins and the earlier is corrected rather than left standing — the [[Marker and Chandra]] revenue cap moved from $2M to $5M, and [[Qwen Model Family]] gained three generations, purely by re-reading. *Leaderboard over self-report:* [[Baidu Unlimited-OCR]] (93.92) and [[OvisOCR2]] (96.58) publish OmniDocBench numbers in their own papers and **appear on no official leaderboard**, while [[PaddleOCR-VL]]-1.6's 96.34 does. Those are different grades of evidence and this table marks which is which.

**One row does not fit the shape of the others.** [[Baidu Unlimited-OCR]] is a *whole-document* model — one request, dozens of pages, one sequence — so its speed cell is not comparable to a pages/second figure and it cannot be dropped into a per-page fan-out ([[Document Fan-Out and Fan-In]]). It was previously left off this table for that reason, which was a mistake: it is MIT, it holds **the highest OmniDocBench score anywhere in this wiki**, and an unstated exclusion reads as an oversight rather than a decision. It is on the table now, with the shape difference stated instead of assumed.

## The three structural facts this table exposes

**1. No single model wins.** The only model with *verified* Hungarian ([[PaddleOCR-VL]]) is the one you **cannot fine-tune** — so its accuracy is a fixed ceiling. Every model you *can* fine-tune has unverified or absent Hungarian. This is not bad luck; it is the shape of the field, and it is why [[Golden Set and Eval Harness]] is Rule 0 rather than a nice-to-have.

**2. Speed and evidence are anti-correlated.** The fastest models ([[GLM-OCR]] 1.86 pages/s, [[LightOnOCR]] 5.71) have the weakest independent standing and no Hungarian. The strongest independently-scored model ([[dots.ocr]]) is **5–16× slower**. Nothing on this table is both fast and proven for Hungarian.

**4. The best benchmark score in the field comes with the least language evidence.** [[Baidu Unlimited-OCR]] scores 93.23–93.92 on OmniDocBench — higher than anything else here — and publishes **no language list and no multilingual benchmark at all**. It is also the one model whose architecture *forecloses* fixing that: [[R-SWA Reference Sliding Window Attention]] requires the reference tokens to stay outside the recursive state, which is why its training froze the [[DeepSeek-OCR]] encoder, which is why it cannot have gained script competence its base lacked. **A benchmark score measured on Chinese and English documents is not evidence about Hungarian**, and this row is the clearest case of that on the table. See [[Benchmark Saturation]].

**3. Licence knocks out a genuine contender.** [[HunyuanOCR]] is the most faithful model anyone measured — the [[Linguistic Crutch and Faithfulness]] outlier — and NOASSERTION makes it unusable commercially without counsel. That is a legal decision, not a technical one, and it should be made deliberately rather than by default. See [[Permissive Licensing Constraints]].

## Probe order

Ordered by **information gained per hour spent**, not by expected win.

**Step 0 — stand up the harness before probing anything.** `vllm serve Qwen/Qwen3.5-9B --served-model-name glm-ocr` gives you a running end-to-end pipeline in one command, because the [[glmocr SDK]] treats the recognition model as a config value. Output quality will be poor — GLM-OCR's fixed prompts meet a different model's dialect — but that is not the point: it makes every row below a **one-environment-variable swap** instead of an integration project. Do this first and the probe order costs hours instead of days.

1. **[[PaddleOCR-VL]]** — the only verified-Hungarian arm. Run it first: it sets the *floor* the fine-tuned candidates must beat to be worth their training cost. Cheap, no training.
2. **Qwen3.5-9B** — the front-runner on odds, and now on measured document competence too. If 201-language text coverage translates to OCR, the search ends here. Highest expected value. Run **9B, not 4B**: a practitioner report finds 0.8B–4B drift into summarising the page instead of transcribing it, which is a faithfulness failure a CER-only check will not catch.
3. **[[dots.ocr]]** — the accuracy ceiling. Slow, so measure quality first and decide about throughput second. Never the reverse.
4. **[[DeepSeek-OCR]] 2** — only as a *fine-tune base*, not as a zero-shot arm. Best adaptation substrate ([[LoRA Fine-Tuning for OCR]]), worst multi-script baseline. **Probe v2, train on v2** — it strictly supersedes v1 on every published number, and the reading-order and repetition improvements are exactly the two failure modes that make v1 unusable raw. Keep v1 only as the historical evidence baseline.
5. **[[Baidu Unlimited-OCR]]** — probe it **only if the page-length distribution justifies it**, and then as a *fidelity* arm, not a throughput one. It answers a different question from every other row: not "can it read Hungarian" but "does one-shot long-horizon parsing preserve cross-page tables and reading order that per-page fan-out breaks". Cheap to answer, and the answer is orthogonal to the language question — so do not let its OmniDocBench score pull it forward in this order.
6. **[[LightOnOCR]]** and **[[GLM-OCR]]** — throughput reference points, not candidates. Run them to know what "fast" costs, so tiering decisions have a real number.

**Figure 2 was read on 2026-08-27, and it eliminated a branch.** Hungarian is not among the 32 languages Qwen3-VL clears 70% on, and the 7 below the threshold are never named — so Qwen3-VL is out as a zero-shot Hungarian arm on either branch. The full roster is in [[Hungarian OCR and the Double Acute Test]].

**What that check did *not* do is disqualify the Qwen family.** Qwen3.5 is a different model with a different training mix, and it is where the sizes are: 0.8B–9B open weights exist **only** in that generation — Qwen3.6 and Qwen3.8 start at 27B, and Qwen3.7 never shipped weights at all. See [[Qwen Model Family]].

## The harness is a separate axis from the model

**Every row above is a config value, not an integration.** The [[glmocr SDK]] survives the rejection of the model it was built for — it keeps [[PP-DocLayout-V3]] layout detection, region fan-out, reading-order assembly and Markdown/JSON formatting, and it calls the recognition model through a plain OpenAI-compatible endpoint. That is the concrete instance of [[Pipeline as Platform, Model as Config]] (Rule 0.5), and it is why this table can be a table at all rather than a set of forks.

The three ways to put a non-GLM model behind it:

| Option | Shape | Use it for |
|---|---|---|
| **A — name alias** | `--served-model-name glm-ocr` on any [[vLLM]] service | **The probe vehicle and smoke test.** Runs immediately; quality degrades because `ResultFormatter` meets a foreign output dialect. Never production. |
| **B — adapter proxy** | ~200-line shim: GLM dialect to the SDK, per-region-type prompts to the model | **The production shape.** The SDK stays a pinned pip dependency; the adapter is the anti-corruption layer every future model slots into. |
| **C — subclass `OCRClient`** | in-process | Only if B's network hop measurably hurts — at region granularity it will not. |

Two consequences for this matrix:

- **The Hungarian question and the harness question are independent.** [[GLM-OCR]] the *model* is disqualified on 8 languages; the *SDK* is unaffected by that and is where the pipeline value sits. Do not let one verdict carry the other.
- **A model that cannot emit `bbox` is not automatically disqualified.** The SDK gets region coordinates from [[PP-DocLayout-V3]] on a normalized 0–1000 scale, independently of the recognition model. That is the fallback for the open `data-bbox` question on [[Qwen Model Family]] 3.5 — see [[The OCR-to-RAG Seam]].

## The route that skips this table entirely

Every row above fills the **recognition slot**. [[ColPali]] does not compete for that slot — it removes the need for it, by indexing page images directly and matching queries against patch embeddings. If the question is "which pages answer this?", late-interaction visual retrieval never needs a character recognised at all.

It belongs on this page for two reasons.

**First, it is subject to the same two decisions as everything else here — and gets different answers.**

| | Recognition slot | [[ColPali]] route |
|---|---|---|
| **Backbone is a config value** | yes, via the [[glmocr SDK]] | yes — ColPali is an *architecture*, and the top checkpoint runs on **Qwen3.5-4B**, the same family as the recognition arm |
| **Permissive licence** | most rows pass | **the original fails** — PaliGemma is Gemma-licensed. Only the ColQwen line is permissive, and `vidore/colqwen2.5-v0.2` has a **live README-vs-card conflict** |
| **Hungarian** | verified on exactly one model, which cannot be fine-tuned | **no variant declares it** — but the failure mode moves from diacritic recognition to *cross-lingual query alignment*, which is a different and probably easier problem |
| **Version churn** | fast | fast, and the **two best checkpoints are community repos**, not `vidore/` — pin the revision hash |

**Second, the backbone convergence is a real architectural option.** The best ColPali checkpoint (ViDoRe 90.9) and the recommended recognition arm both run on Qwen3.5-4B/9B. One model family serving both slots means one set of serving patterns, one VRAM profile family, and one Gated-DeltaNet caveat instead of two. That is worth weighing before treating the two halves as independent procurement decisions.

**What it does not change:** the adoption threshold is still the one in [[The OCR-to-Text Boundary Limit]] — if OCR already answers >~95% of table and chart questions correctly, defer it — and the ~1,030 vectors per page still make token pooling mandatory rather than optional. Full detail on [[ColPali]].

## What must be true before this table means anything

Every ranking here is a **prior**, not a result. It becomes a decision only against a golden set: 200–500 pages from your own corpus, diacritic-specific error rate broken out separately from overall CER, `hu_HU` hunspell validation, and Latin-2/CP-1250 mojibake detection. See [[Golden Set and Eval Harness]] and [[Open Inputs and Corpus Profile]].

## Related

[[Coverage and Exclusion Register]] · [[Hungarian OCR and the Double Acute Test]] · [[Tiered Page Routing]] · [[Benchmark Saturation]] · [[Pipeline as Platform, Model as Config]] · [[Cost per Page Model]] · [[Permissive Licensing Constraints]]
