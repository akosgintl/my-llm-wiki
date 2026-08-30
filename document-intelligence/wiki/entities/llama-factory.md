---
aliases: ["LLaMA-Factory"]
tags: [fine-tuning, lora, tool]
sources: [ocr-language-models-deployment-critical-pass-2.md, ocr-arxiv-github-technical-review.md, Deep Technical Review - Open-Source OCR - Visual Document Understanding Models - Missed Details and Pipeline Implications.md]
created: 2026-08-27
updated: 2026-08-27
---

# LLaMA-Factory

The fine-tuning framework with the **official [[GLM-OCR]] recipe**, shipped in-repo at `examples/finetune/README.md` (added 2026-02-12).

## The recipe

`lora_rank: 8, lora_target: all` for LoRA; full SFT freezes the vision tower and projector and trains the LM only. That freeze-the-encoder shape matches both [[Unsloth]]'s guidance and [[Baidu Unlimited-OCR]]'s own continue-training recipe — Hungarian is a language problem, not a vision problem.

## The undocumented risk it carries

The finetune README **never mentions the MTP / NextN head**. Whether `llamafactory-cli export` merge yields a functional [[Multi-Token Prediction and Speculative Decoding]] head is undocumented in the repo, the paper, or any issue.

This is not "not found" — it is genuinely undocumented, and it puts GLM-OCR's ~50% speculative-decoding throughput gain at risk after any fine-tune. It must be validated empirically under `--speculative-config method mtp`; if it fails, either serve base+adapter unmerged or budget for the throughput loss.

## Related

[[Unsloth]] · [[GLM-OCR]] · [[LoRA Fine-Tuning for OCR]] · [[Multi-Token Prediction and Speculative Decoding]]
