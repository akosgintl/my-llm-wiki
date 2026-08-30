---
aliases: ["DeepSeek AI"]
tags: [organization]
sources: [Open-Source OCR per Visual Document Understanding Models for Self-Hosted AKS Pipelines - A Comparative Investigation (August 2026).md, ocr-arxiv-github-technical-review.md, Deep Technical Review - Open-Source OCR - Visual Document Understanding Models - Missed Details and Pipeline Implications.md]
created: 2026-08-27
updated: 2026-08-27
---

# DeepSeek AI

Publisher of [[DeepSeek-OCR]] (Oct 2025) and DeepSeek-OCR 2 (Jan 2026), and — indirectly — of [[Baidu Unlimited-OCR]], which is a continue-trained fork of their checkpoint.

Their contribution to the field is the **[[Optical Compression]] thesis**: that a document page can be encoded into far fewer vision tokens than its text tokens, with quantified degradation (~97% at <10× compression, ~60% at 20×). That framing now shapes how every compact document VLM budgets resolution.

Two liabilities travel with the lineage and are worth naming as organizational facts, not just model facts:

1. **The AGPL PyMuPDF dependency** (GitHub Issue #223) disputes the repo's MIT claim and is a genuine commercial blocker for anything that vendors their scripts. See [[Permissive Licensing Constraints]].
2. **[[Linguistic Crutch and Faithfulness]]** — an independent critique (arXiv 2601.03714) shows the models lean on language priors, "correcting" visually clear but semantically odd text. That behavior is inherited by anything continue-trained from the checkpoint.

Offsetting both: DeepSeek-OCR has the best fine-tuning tooling of any candidate via [[Unsloth]], which makes it the strongest *trainable* arm even where it is not the strongest off-the-shelf arm.

## Related

[[DeepSeek-OCR]] · [[Baidu Unlimited-OCR]] · [[Optical Compression]] · [[Unsloth]]
