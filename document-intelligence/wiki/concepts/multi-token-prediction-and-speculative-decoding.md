---
aliases: ["Multi-Token Prediction and Speculative Decoding"]
tags: [serving, performance, architecture, optimization]
sources: [ocr-arxiv-github-technical-review.md, Deep Technical Review - Open-Source OCR - Visual Document Understanding Models - Missed Details and Pipeline Implications.md, ocr-serving-recipes-runpod-v2.md, ocr-vdu-complete-study.md]
created: 2026-08-27
updated: 2026-08-27
---

# Multi-Token Prediction and Speculative Decoding

The one place in this architecture where speculative decoding is worth enabling — and one open risk attached to it.

## The mechanism

[[GLM-OCR]] is trained with **Multi-Token Prediction**: k shared-parameter auxiliary heads modeling different future offsets, kept enabled through SFT. Verbatim from the paper:

> *"GLM-OCR is trained to predict ten tokens per step and generates 5.2 tokens per decoding step on average at inference time, bringing approximately 50% throughput improvement."*

The elegant part: **the same head doubles as the speculative draft model at serving time.** No separate draft model, no extra weights.

```bash
# vLLM
--speculative-config '{"method": "mtp", "num_speculative_tokens": 3}'

# SGLang
SGLANG_ENABLE_SPEC_V2=1 ... --speculative-algorithm NEXTN \
  --speculative-num-steps 3 --speculative-eagle-topk 1 --speculative-num-draft-tokens 4
```

## Where it applies — and where it does not

**Applies:** GLM-OCR, and the Qwen3.5/3.6 lineage (MTP-trained; Qwen3.6 reports 1.4–2.2×; MTP is supported in [[vLLM]] for the Qwen3-Next lineage).

**Does not apply:** everything else. **Do not bolt draft-model speculation onto the other models** — OCR output is already cheap per token on MoE decoders, so there is nothing to win. It is L4 of the [[Parallelism Stack for OCR Serving]], and it is a narrow layer by design.

## The open risk: does MTP survive a LoRA merge?

**Undocumented in the repo, the paper, and every issue.** The [[LLaMA-Factory]] finetune README (2026-02-12) never mentions the MTP/NextN head. Whether `llamafactory-cli export` produces a functional MTP head after merging is genuinely unknown — not merely unfound.

This matters because a Hungarian fine-tune is likely mandatory for GLM-OCR, so the merge is not hypothetical.

**Checked 2026-08-27 — still vendor-undocumented, but two things narrowed:**

1. The official vLLM recipe confirms the MTP layers are **built into the published checkpoint**, not a side-loaded module (*“GLM-OCR model include built-in Multi-Token Prediction (MTP) layers”*), and serves them with `--speculative-config.method mtp --speculative-config.num_speculative_tokens 1`. Note the **official recipe uses 1 speculative token, not 3** — start there and tune upward. It also requires `transformers >= 5.0.0`.
2. Because the heads are in-checkpoint weights, a LoRA run that targets only decoder attention and MLP projections leaves them **untrained but structurally intact** — so the realistic failure is a *degraded acceptance rate*, not a crash. If it degrades there is a documented mitigation: **gated LoRA** (MTP-GLoRA, arXiv 2507.11851) activates adapters only on MTP token positions so base next-token behaviour is preserved, and vLLM's *speculators* path supports finetuning the existing MTP layers and stitching the weights back.

**The cheap test:** after `llamafactory-cli export`, serve the merged checkpoint with MTP enabled and read vLLM's **draft acceptance rate** from `/metrics`. If it collapses toward zero the head did not survive — drop to `num_speculative_tokens 0` and accept the ~50% throughput loss, or re-run with gated LoRA. This is a one-hour measurement, not a research project. See [[LoRA Fine-Tuning for OCR]].

**Mitigation:**

1. Empirically verify the merged checkpoint still exposes a working MTP head under `--speculative-config method mtp` **before production serving**.
2. If it does not: either serve base + adapter unmerged (losing the merge's simplicity) or budget for the ~50% throughput loss and scale replicas via KEDA instead.

Treat it as a fine-tune QA item with an explicit gate, not an assumption. See [[LoRA Fine-Tuning for OCR]].

## Related

[[GLM-OCR]] · [[Parallelism Stack for OCR Serving]] · [[LoRA Fine-Tuning for OCR]] · [[Qwen Model Family]] · [[SGLang]]
