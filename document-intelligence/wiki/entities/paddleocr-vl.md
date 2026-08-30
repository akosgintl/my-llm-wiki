---
aliases: ["PaddleOCR-VL"]
tags: [ocr, vlm, document-parsing, multilingual, model]
sources: [Open-Source OCR per Visual Document Understanding Models for Self-Hosted AKS Pipelines - A Comparative Investigation (August 2026).md, ocr-serving-recipes-runpod-v2.md, ocr-language-models-deployment-critical-pass-2.md, ocr-vdu-complete-study.md, ocr-arxiv-github-technical-review.md, Deep Technical Review - Open-Source OCR - Visual Document Understanding Models - Missed Details and Pipeline Implications.md, Local PDF-Markdown and Document Parsing - State of the Art (August 2026).md, tiered-pdf-pipeline-architecture.md]
created: 2026-08-27
updated: 2026-08-27
---

# PaddleOCR-VL

[[Baidu]]'s 0.9B document VLM, Apache-2.0. NaViT-style dynamic-resolution encoder + ERNIE-4.5-0.3B decoder, driven by a [[PP-DocLayout-V3]] front stage. Papers: v1 arXiv 2510.14528 · v1.5 arXiv 2601.21957 · v1.6 arXiv 2606.03264.

## Versions

| Version | Date | OmniDocBench | Notes |
|---|---|---|---|
| v1 | 2025 | 92.56 / 92.86 | 109 languages |
| v1.5 | 2026-01-29 | 94.5 (v1.5) / 94.87–94.93 (v1.6) | + seal recognition, text spotting, distortion robustness; +Tibetan +Bengali → 111 languages |
| v1.6 | 2026-05-28 | **96.33** (v1.6) | Architecture unchanged from 1.5 — "zero-cost migration"; gains from region-aware data engineering + progressive CPT→SFT→RL post-training |

Text edit 0.033–0.035, formula CDM up to 97.5, Table TEDS ~94.8. New SOTA on Real5-OmniDocBench (in-the-wild distortions) at ~92.05. The table training corpus is 5M+ image-table pairs built by auto-annotation, annotation mining and synthesis, which explains the TEDS lead.

## Hungarian: confirmed

Hungarian is **explicitly listed** in the official Latin-group language list. Classic PP-OCRv5's Latin recognizer (32 languages) also lists Hungarian. This is the only verified-Hungarian row on the [[Hungarian Model Decision Matrix]], and it makes PaddleOCR-VL a legitimate no-fine-tune arm.

### Re-verified from the primary source, 2026-08-27

Read directly from the PDF of **arXiv 2510.14528, Appendix B "Supported Languages", Table A1** — a **text table, not a raster image**, so this is a hard read rather than an eyeball:

> *"French, German, Afrikaans, Italian, Spanish, Bosnian, Portuguese, Czech, Welsh, Danish, Estonian, Irish, Croatian, Uzbek, **Hungarian**, Serbian (Latin), Indonesian, Occitan, Icelandic, Lithuanian, Maori, Malay, Dutch, Norwegian, Polish, Slovak, Slovenian, Albanian, Swedish, Swahili, Tagalog, Turkish, Latin, Azerbaijani, Kurdish, Latvian, Maltese, Pali, Romanian, Vietnamese, Finnish, Basque, Galician, Luxembourgish, Romansh, Catalan, Quechua"*

That is the **Latin** category: **47 languages** of the model's 109. Membership is settled — and it is the strongest language evidence any model on the matrix has.

### What the re-check also found, and it matters

**1. There is no Hungarian-specific accuracy number anywhere.** The paper's per-language claim points at Table 6a, but **Table 6a reports by script group, not by language.** PaddleOCR-VL's cell reads **Latin = 0.013 edit distance** — the best score in the table, across all ten groups — but that single figure aggregates all 47 Latin-group languages, from French and German to Romansh and Quechua.

**A Latin-group mean dominated by French, German and Spanish is exactly where a Hungarian double-acute failure would be invisible.** This is the whole argument of [[Hungarian OCR and the Double Acute Test]], applied to the one model that passes the membership test. Membership is verified; **accuracy on ő and ű is not, and cannot be read off this table.**

**2. The measurement is vendor data.** The 0.013 comes from Baidu's self-built *In-house-OCR* set (107,452 line-level samples), not an independent benchmark. See [[Benchmark Saturation]].

**3. The roster belongs to v1 and was never restated.** The 109-language list is from the October 2025 paper. The **1.6 technical report (arXiv 2606.03264) does not mention languages at all** — it is a region-refinement and RL post-training paper. So the roster is *inherited* by 1.5 and 1.6 rather than reconfirmed, and RL post-training is in principle capable of shifting multilingual behaviour. No evidence exists either way; treat the inheritance as reasonable but unverified.

**Net:** this row keeps its ✅, and the ✅ means *"Hungarian is a declared, trained language"* — not *"Hungarian accuracy is measured"*. The distinction is exactly what the golden set exists to close. See [[Golden Set and Eval Harness]].

## The fine-tuning contradiction

Official docs FAQ, verbatim: *"Currently, we do not support fine-tuning of the model, but it is a high-priority feature and will be released soon."* Earlier study material rated Paddle fine-tuning as merely "less turnkey" — in fact **no supported path exists today**. Consequence: PaddleOCR-VL is the *ceiling-fixed* arm. If its Hungarian diacritics disappoint you swap models rather than tune it, and the fine-tune arms shift to [[Qwen Model Family]] and [[DeepSeek-OCR]]. See [[LoRA Fine-Tuning for OCR]].

## Deployment gotchas

- **Hard requirement: NVIDIA compute capability ≥ 8.0 (Ampere+).** T4/V100 OOM or time out. Blackwell needs a special path.
- **You must run the full two-stage pipeline**, not just the VLM, or hallucination increases and paper accuracy does not reproduce: `paddleocr doc_parser --vl_rec_backend vllm-server --vl_rec_server_url http://HOST:PORT/v1`.
- **Chart recognition is OFF by default** — set `use_chart_recognition=True` or you silently compare models with a capability missing.
- FastDeploy is Baidu's fastest backend (~2.0 pages/s on A100 vs ~1.38 for [[vLLM]]), but vLLM is the more portable choice. OpenVINO path via `paddleocr_vl_ov`.
- Table target format is **OTSL**, not HTML. Reading order comes from the layout stage's Global-Pointer mechanism — consume it rather than re-sorting by y-coordinate.

## Verdict

Best compact model for multilingual and in-the-wild (scanned, photographed, skewed) documents, and the most robust on vertical/messy text among the compact models (MORE 87.96, second only to [[HunyuanOCR]] and Gemini). The CPU layout step can bottleneck GPU utilization under concurrency — see [[Layout Stage Economics]].

## Related

[[GLM-OCR]] · [[dots.ocr]] · [[MinerU]] · [[PP-DocLayout-V3]] · [[OmniDocBench]] · [[Tiered Page Routing]]
