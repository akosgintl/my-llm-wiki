---
aliases: ["GLM-OCR"]
tags: [ocr, vlm, document-parsing, model]
sources: [Open-Source OCR per Visual Document Understanding Models for Self-Hosted AKS Pipelines - A Comparative Investigation (August 2026).md, ocr-serving-recipes-runpod-v2.md, ocr-api-call-strategy.md, ocr-language-models-deployment-critical-pass-2.md, ocr-vdu-complete-study.md, ocr-arxiv-github-technical-review.md, Deep Technical Review - Open-Source OCR - Visual Document Understanding Models - Missed Details and Pipeline Implications.md, Local PDF-Markdown and Document Parsing - State of the Art (August 2026).md]
created: 2026-08-27
updated: 2026-08-27
---

# GLM-OCR

Compact document VLM from [[Zhipu AI]] (Feb 2026; tech report arXiv 2603.10910). 0.9B total: a 0.4B CogViT visual encoder plus a 0.5B GLM decoder. MIT weights; the bundled [[PP-DocLayout-V3]] layout model is Apache-2.0.

## Architecture

- Two-stage: [[PP-DocLayout-V3]] detects regions, the VLM recognizes each region crop in parallel. See [[Two-Stage vs End-to-End Document Parsing]].
- [[Multi-Token Prediction and Speculative Decoding]]: trained to predict ~10 tokens/step, realizing ~5.2 at inference (~50% throughput gain); the same head serves as a vLLM speculative draft.
- Trained with GRPO RL. Reward table (paper Table 2): Normalized Edit Distance for text, CDM + structural validity for formulas, TEDS + tag-closure for tables, field-level F1 + JSON-parse validation for KIE, plus **global repetition-ratio and malformed-structure penalties**. No closed-form equation is published. This reward menu is a copyable template for a format-native fine-tune.
- Cross-modal connector with token downsampling + SwiGLU. **Ratio resolved 2026-08-27 from `config.json`, not the paper: `patch_size: 14`, `spatial_merge_size: 2`** — the connector merges each **2×2 patch block into one visual token, a 4× reduction**, so one effective visual token covers a **28×28 px region** (`image_size: 336`; vision `hidden_size` 1024 → `out_hidden_size` 1536). Use 28 px, not 14, when costing a region request in [[Layout Stage Economics]].

## Numbers

- OmniDocBench v1.5: 94.62 overall (v1.6: 95.15–95.22). Table TEDS 93.96, formula CDM 93.90, UniMERNet 96.5.
- KIE: Nanonets-KIE 93.7, Handwritten-KIE 86.1. In-house: seals 90.5 (vs dots.ocr 63.0), receipts 94.5.
- 1.86 PDF pages/s, 0.67 images/s — fastest in class. ~2.9–3 GB VRAM at FP16 (~0.7 GB INT4).
- Independent signal is much weaker: [[OCR Arena]] #24 (ELO 1347), [[olmOCR-Bench]] ~75.2 with a Long/Tiny-text score of only ~35.7. See [[Multi-Script OCR Benchmarks]].

## Prompts and serving

Byte-exact prompt strings: `{"text":"Text Recognition:","formula":"Formula Recognition:","table":"Table Recognition:"}`, with `<image>` placed **before** the text prompt. KIE has no canonical string — it is a free-form strict-JSON-schema instruction. Custom prompts are allowed (training data includes `<image>Code Generation:`).

Served on [[vLLM]] with `--speculative-config '{"method":"mtp","num_speculative_tokens":3}'`, or [[SGLang]] with `NEXTN`. Serving the bare VLM without the [[glmocr SDK]] pipeline skips layout analysis and does not reproduce paper accuracy.

## The Hungarian disqualification

Paper footnote 2 lists exactly **8 languages** (zh, en, fr, es, ru, de, ja, ko). Hungarian is **not** among them; the "100+ languages" claim is unverified marketing absent from paper, README and model card. Hands-on tests report hangs/gibberish on Hindi, Arabic, Polish and French. This single fact demoted GLM-OCR from default workhorse to niche tool — see [[Hungarian OCR and the Double Acute Test]]. The [[glmocr SDK]] stays; the model is replaced.

## Open risks

- **MTP survival under LoRA merge is undocumented** in repo, paper or issues. The LLaMA-Factory finetune README (2026-02-12, `lora_rank: 8, lora_target: all`) never mentions the MTP/NextN head. Must be validated empirically or the ~50% speculative gain silently disappears. See [[LoRA Fine-Tuning for OCR]].
- Prompt rigidity (small fixed prompt set, no free-form Q&A); two-stage error propagation; stochastic whitespace/line-break variation.

## Related

[[PaddleOCR-VL]] · [[glmocr SDK]] · [[HunyuanOCR]] · [[Qwen Model Family]] · [[OmniDocBench]]
