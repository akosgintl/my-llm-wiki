---
aliases: ["Baidu Unlimited-OCR"]
tags: [ocr, vlm, document-parsing, long-context, model]
sources: [Open-Source OCR per Visual Document Understanding Models for Self-Hosted AKS Pipelines - A Comparative Investigation (August 2026).md, ocr-serving-recipes-runpod-v2.md, ocr-api-call-strategy.md, ocr-vdu-complete-study.md, ocr-arxiv-github-technical-review.md, Deep Technical Review - Open-Source OCR - Visual Document Understanding Models - Missed Details and Pipeline Implications.md]
created: 2026-08-27
updated: 2026-08-27
---

# Baidu Unlimited-OCR

Released 2026-06-22 by [[Baidu]] (arXiv 2606.23050, "Unlimited OCR Works"). MIT weights and code. The only candidate in the field that parses **dozens of pages in a single request**.

## How it was made

Continue-trained from the [[DeepSeek-OCR]] checkpoint: ~4,000 steps, **DeepEncoder frozen, decoder only**, ~2M document samples with a **9:1 single-page to multi-page split** where multi-page samples were built by simple concatenation, on 8×16 A800. This recipe is the directly copyable template for a long-horizon Hungarian variant — see [[LoRA Fine-Tuning for OCR]].

## R-SWA — the whole trick

Every decoder attention layer is replaced with **Reference Sliding Window Attention**. Each generated token attends to:

1. **all reference tokens** (visual + prompt, length m, fixed at inference start) held statically *outside* the state transitions, and
2. only the preceding **n = 128** output tokens.

The KV cache is a queue of capacity m + n; each new token evicts the oldest *output* token. Cache ceiling = L·m + n instead of O(T). See [[R-SWA Reference Sliding Window Attention]].

Why not vanilla SWA or linear attention: there, visual tokens get pulled into recursive state updates and image features **progressively blur** as pages advance. Excluding reference tokens from the sliding state is the entire innovation — and also why the fine-tune recipe freezes the encoder.

## Numbers

- OmniDocBench 93.23 (v1.5, **+6.22** over its DeepSeek-OCR baseline, beating DeepSeek-OCR-2's 89.17 in the same table) / 93.92 (v1.6).
- 40+ pages in one call at edit distance <0.11. Anti-loop metric: Distinct-20 = 96.08%, Distinct-35 = 96.90%.
- ~35% faster than DeepSeek-OCR at 6,000 generated tokens (TPS 7,847 vs 5,823), gap widening with length.

## Language coverage — the gap this page carried until 2026-08-27

**Checked against the model card and the repo: there is no language roster.** The card carries a bare
`multilingual` tag, enumerates nothing, and never mentions Hungarian. No multilingual benchmark result
is published either — not OmniDocBench multilingual, not socOCRbench, not GlotOCR, not MDPBench. The
only leaderboard number on the card is ParseBench (mean 46.17, text content 86.81, text formatting 0.97).
Community reports cover Cyrillic and handwritten maths; nothing on Central European diacritics.

**And there is a structural reason to expect the coverage to be no better than its base.** The model
is continue-trained from [[DeepSeek-OCR]] with the **DeepEncoder frozen** — only the decoder moved,
over ~2M samples that were 90% single-page and built largely by concatenation. Freezing the encoder is
not incidental: it is *why* [[R-SWA Reference Sliding Window Attention]] works, because reference
tokens must stay outside the recursive state. But the same choice means the model **cannot have gained
visual script competence its base did not have**, and [[DeepSeek-OCR]] v1 is the weakest multi-script
model in the field — below Tesseract v5 on socOCRbench.

So the honest reading is: **long-horizon capability was bought on top of a weak multilingual
foundation, and the mechanism that delivers it forecloses fixing that foundation by continued
pre-training.** Adapting it to Hungarian means a decoder-side fine-tune on Hungarian pages, not a
better checkpoint. That is possible — the card now documents **ms-swift** training support — but it
is the same work as adapting DeepSeek-OCR, with the same encoder ceiling. See
[[Hungarian OCR and the Double Acute Test]] and [[LoRA Fine-Tuning for OCR]].

## Token budgeting

`max_length = 32768` covers **prompt + reference + output together**. The naive heuristic "pages × ~800 output tokens" must additionally subtract ~300 vision tokens per page and the prompt. Practical ceiling ~30–40 text pages, fewer for dense tables; chunk at ~25–30 pages with 1-page overlap. See [[Document Fan-Out and Fan-In]].

## Serving

Two modes: **gundam** (base_size 1024, image_size 640, crop, fast single-image, `window_size=128`) and **base** (image_size 1024, no crop, multi-page, `window_size=1024`). The prompt must literally begin with `<image>` (`<image>document parsing.` / `<image>Multi page parsing.`). Raw output carries `<|ref|>…<|/ref|>` text and `<|det|>…<|/det|>` coordinate boxes — unwrap ref, drop det for clean markdown.

- [[vLLM]]: dedicated image `vllm/vllm-openai:unlimited-ocr` plus the unlimited_ocr n-gram logits processor, caches off.
- [[SGLang]]: `--attention-backend fa3` (**FlashAttention-3, Hopper-only**), `--page-size 1`, `--enable-custom-logit-processor`, `--disable-overlap-schedule`, using the **dev-build SGLang wheel bundled in the repo** — pin it in your image, do not `pip install sglang` fresh.

**Never mix it into a page-model load-balancer pool.** A 20-minute document request behind the same LB as 2-second region requests wrecks every latency-based balancing signal. Separate pool, separate queue, in-flight = max_num_seqs exactly. See [[Load Balancing Inference Pools]].

## Limitations

Not truly unlimited — 32K bounds the prefill. Multi-page uses base mode only, so very small text can be missed. No bounding boxes or per-word confidence. Likely inherits DeepSeek's repetition tendency and [[Linguistic Crutch and Faithfulness]] risk. Independent production evaluation is thin. Roadmap: 128K-context training and a "prefill pool" that fetches chunks on demand — so do not over-engineer chunking now.

## Related

[[DeepSeek-OCR]] · [[R-SWA Reference Sliding Window Attention]] · [[MIG and GPU Sharing]] · [[vLLM]] · [[Hungarian Model Decision Matrix]] · [[Document Fan-Out and Fan-In]]
