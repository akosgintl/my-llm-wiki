---
aliases: ["LoRA Fine-Tuning for OCR"]
tags: [fine-tuning, lora, hungarian, methodology]
sources: [ocr-language-models-deployment-critical-pass-2.md, ocr-vdu-complete-study.md, ocr-arxiv-github-technical-review.md, Deep Technical Review - Open-Source OCR - Visual Document Understanding Models - Missed Details and Pipeline Implications.md]
created: 2026-08-27
updated: 2026-08-27
---

# LoRA Fine-Tuning for OCR

**Fine-tune only if the probe demands it.** Run the bake-off first — if the best off-the-shelf CER is already low-single-digit with clean diacritics, ship it and move on. See [[Golden Set and Eval Harness]].

## The recipe

**Freeze the vision encoder, LoRA the decoder.** Hungarian is a *language* problem, not a vision problem, and three independent sources converge on this shape:

- [[Baidu Unlimited-OCR]] was **made** this way: 4,000 steps continue-trained from [[DeepSeek-OCR]], DeepEncoder frozen, decoder only, ~2M samples on 8×16 A800.
- [[LLaMA-Factory]]'s official [[GLM-OCR]] recipe freezes the vision tower and projector.
- [[Qwen Model Family]] training Stage-0 updates **only the MLP merger** with encoder and LLM frozen — which also makes the merger a legitimate extra LoRA target.

Mechanics: LoRA r=16–64 on the decoder (+projector), 1–3 epochs, fits **one 24 GB card** under [[Unsloth]]. Merge the adapter before serving so the [[vLLM]] recipes stay untouched.

## Tooling maturity decides more than architecture

[[Unsloth]] for [[DeepSeek-OCR]] (★★★, but **use their modified checkpoint** — the stock one does not run on current transformers) and Qwen3.5 (★★★, day-1). [[LLaMA-Factory]] for GLM-OCR (★★). **[[PaddleOCR-VL]] is officially unsupported (★)** — which removes it as a fine-tune target entirely and strengthens the Qwen/DeepSeek arms.

## Dataset economics

| Scale | Outcome |
|---|---|
| 2–5K page/region pairs | meaningful gains |
| **10–30K** | solid production adaptation |
| 50–100K+ | robustness across fonts and degradation |

**Sourcing beats sizing** — see [[Born-Digital Self-Labeling]] for the two cheats that make 10–30K pairs cost roughly zero annotation hours.

## The three rules that make or break it

**1. Anti-forgetting.** Keep **10–30% of the original multilingual/table/formula distribution** in the training mix, or the fine-tune will destroy the very table and layout skills that justified choosing the base model.

**2. Format-native targets — the move that dissolves two problems at once.** Since a Hungarian LoRA is likely anyway, make the training targets *the pipeline's expected output conventions* (region crop → text / strict HTML tables / bare LaTeX). **One training run buys the language AND the output contract**, the adapter shim shrinks toward identity, and conformity is guaranteed by weights instead of prompt engineering. See [[Pipeline as Platform, Model as Config]].

**3. Gate every checkpoint on the held-out golden set with the diacritic metric, not CER alone.**

## Reward design worth copying

Three published objective designs, all reusable:

- **GRPO metric rewards** ([[GLM-OCR]]): NED for text, CDM for formulas, TEDS for tables, field-F1 for KIE, plus explicit repetition and malformed-structure penalties. The anti-loop behavior is *trained in*, not emergent.
- **RLVR with unit-test rewards** ([[olmOCR]]): binary verifiable checks aggregated to a page pass rate. If you already run region-level CI fixtures, RLVR turns those fixtures into training rewards.
- **Data engineering alone** ([[MinerU]]2.5-Pro): +5 points on OmniDocBench v1.6 with an *identical* architecture. The strongest published evidence that the data plan beats architecture chasing.

## Two QA items

- **Does MTP survive the merge?** Undocumented for GLM-OCR. Verify empirically or lose ~50% throughput. See [[Multi-Token Prediction and Speculative Decoding]].
- **The Unsloth Persian numbers are a vendor demo on a different script.** 88.26% absolute CER improvement with most gains after 60 steps is proof the *mechanism* works, not a promise about your corpus. Dataset-size tiers are engineering heuristics, not literature-backed for Hungarian.

## Related

[[Born-Digital Self-Labeling]] · [[Golden Set and Eval Harness]] · [[Unsloth]] · [[Hungarian OCR and the Double Acute Test]] · [[Pipeline as Platform, Model as Config]]
