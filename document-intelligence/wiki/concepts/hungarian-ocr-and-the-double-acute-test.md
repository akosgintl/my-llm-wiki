---
aliases: ["Hungarian OCR and the Double Acute Test"]
tags: [hungarian, evaluation, ocr, multilingual]
sources: [ocr-language-models-deployment-critical-pass-2.md, ocr-vdu-complete-study.md, Deep Technical Review - Open-Source OCR - Visual Document Understanding Models - Missed Details and Pipeline Implications.md, ocr-arxiv-github-technical-review.md, tiered-pdf-pipeline-architecture.md]
created: 2026-08-27
updated: 2026-08-27
---

# Hungarian OCR and the Double Acute Test

The single fact that re-ranked every model in the project: **the corpus is 80% Hungarian.**

It arrived after the supply side had already been designed — *"we designed the supply side before interrogating the demand side, and the demand side promptly invalidated a core choice."* One sentence demoted [[GLM-OCR]] from default workhorse to niche tool.

## The ő/ű test

Hungarian's **double-acute accents (ő, ű) are unique to the language** and are the canonical OCR failure: silent substitution of ö/ü, or of õ/û.

Why it needs its own metric: plain CER barely registers it — one character among thousands — while it corrupts every downstream field and search index. So the probe protocol measures three numbers, and **the third is the tiebreaker**:

1. CER
2. Word accuracy
3. **Diacritic-specific error rate** — á/é/í/ó/ú/ö/ü/ő/ű confusions counted separately

Protocol: 30–50 pages from the *real* corpus (mixed born-digital and scanned), all candidates via their cheapest inference path. Two days of work. See [[Golden Set and Eval Harness]].

## Status board

| Model | Hungarian status | Verdict |
|---|---|---|
| [[GLM-OCR]] | 8 core languages, none Hungarian; breaks on adjacent Latin languages (Polish) | ❌ Disqualified as default (the SDK stays) |
| [[PaddleOCR-VL]] | **Verified 2026-08-27 from the primary PDF** — named in the 47-language Latin group of the 109 (arXiv 2510.14528, Table A1, a text table). **But the only accuracy figure is group-level** — Latin 0.013 across 47 languages, on vendor data. Membership proven, ő/ű accuracy not | ✅ But *not fine-tunable* — the ceiling-fixed arm |
| [[dots.ocr]] | 100+ languages, 2nd on GlotOCR mid-resource | ✅ Serious candidate |
| [[HunyuanOCR]] | Top open model on MORE; most faithful | ✅ Candidate (Hungarian membership in MORE-149 unconfirmed) |
| [[DeepSeek-OCR]] | ~100 languages claimed; near-bottom on multi-script — but Hungarian is Latin-script, so the floor is higher | ⚠️ Probe it; strongest fine-tune base |
| Qwen3-VL | **Verified negative (2026-08-27)** — Figure 2 read off the PDF: **Hungarian is not among the 32 languages that clear 70%**. See the roster below | ❌ Eliminated as a zero-shot Hungarian arm |
| **Qwen3.5-9B / 4B** | 201 languages — but that is a **text** claim, not OCR, and the generation publishes **no per-language OCR breakdown at all** | ✅ **Front-runner** on odds; document competence now proven, Hungarian still not |
| [[LightOnOCR]] | **Verified negative (2026-08-27)** — card declares 11 languages (`en fr de es it nl pt sv da zh ja`), no CEE language at all | ❌ Not a Hungarian candidate; keep as throughput baseline and fine-tune base |

The Qwen3.5-Omni Fleurs top-59 *does* list Hungarian — but that is **speech, not OCR**. Do not let it substitute for a probe.

### The Qwen3-VL Figure 2 roster, read in full

Read from arXiv 2511.21631, page 17, on **2026-08-27**. The figure is a raster image with no text layer; these labels were rendered at 600 dpi and read off the axis. Ordered by accuracy, ascending:

> Romanian ~71 · Swahili ~71 · Russian ~72 · Hindi ~72 · Hebrew ~72 · **Polish ~74** · Cebuano ~74 · Italian ~78 · German ~78 · Vietnamese ~79 · **Ukrainian ~82** · Uzbek ~83 · Spanish ~83 · French ~83 · Portuguese ~83 · Japanese ~84 · Turkish ~86 · Kazakh ~86 · Korean ~87 · Arabic ~87 · Persian ~88 · Urdu ~89 · Finnish ~91 · Dutch ~92 · Norwegian ~92 · **Czech ~92** · Greek ~92 · Thai ~93 · Indonesian ~95 · Danish ~97 · **Serbian ~97** · Swedish ~98

**Exactly 32 bars — the figure plots only the languages above the 70% threshold, not all 39 supported.** The remaining 7 are named nowhere: the string "Hungarian" does not appear anywhere in the paper, and there is no other language list, figure or table in it.

Three things this settles:

1. **Hungarian is definitively not in the ≥70% set.** Not an inference — all 32 labels were read.
2. **The best case is still a bad case.** If Hungarian is in the 39 at all, it is one of the 7 unnamed languages *below* 70% — the level the paper itself calls the threshold for "practical for real-world usability". Either branch eliminates Qwen3-VL as a zero-shot Hungarian recognizer.
3. **This is not a regional gap.** Polish, Czech, Serbian, Ukrainian and Romanian are all present. Hungarian is specifically absent — a targeted hole, not an oversight about Central Europe.

**One suggestive pattern, stated as inference and not as a paper claim:** the bottom of the chart is Romanian (~71) and Polish (~74), while Swedish and Danish sit at 97–98. The Latin-script languages with dense diacritics cluster at the weak end. That is the shape this page's whole thesis predicts — and it is a reason to expect Hungarian to be *hard*, not merely *unlisted*.

> **The full per-model comparison — licence, fine-tunability, cost and independent evidence on one table — lives in [[Hungarian Model Decision Matrix]].**

## Beyond model choice

**Fine-tuning.** If the probe demands it, Hungarian is a *language* problem, not a vision problem: freeze the encoder, LoRA the decoder. Latin plus new diacritics is strictly easier than a new script. See [[LoRA Fine-Tuning for OCR]].

**Validation must be Hungarian-aware.** A dictionary-word-ratio garble check needs the `hu_HU` hunspell dictionary — an English wordlist would flag every correct Hungarian page as garbage and escalate the whole corpus to the GPU. Add a mojibake pattern for Latin-2 / CP-1250, since older Hungarian PDFs often carry broken `õ`-for-`ő` embedded text that should skip the fast tier entirely. Set Tesseract `lang=hun` explicitly, never autodetect. See [[Tiered Page Routing]].

**Deterministic validators are the best-calibrated confidence signal available** for Hungarian business documents: adóazonosító/adószám checksums, IBAN check digits, date-format and VAT-rate sanity. Regex-cheap and perfectly calibrated for exactly the fields that matter most. See [[Confidence Engineering]].

**Post-OCR diacritic restoration is tempting and dangerous.** *kör* versus *kőr* is genuinely a language-model problem — but an unrestrained text LLM will fluently "fix" names, amounts and IDs. If used: diacritic-only edit constraint, edit-distance guard, and never on numeric or ID fields.

**On the retrieval side**, the same principle holds in reverse: use multilingual embeddings, not translation, and do not reach for a Hungarian-only embedding model. See [[BGE-M3]].

## Related

[[Golden Set and Eval Harness]] · [[LoRA Fine-Tuning for OCR]] · [[Confidence Engineering]] · [[BGE-M3]] · [[huBERT]] · [[Tiered Page Routing]]
