---
aliases: ["Zhipu AI"]
tags: [organization]
sources: [Open-Source OCR per Visual Document Understanding Models for Self-Hosted AKS Pipelines - A Comparative Investigation (August 2026).md, ocr-arxiv-github-technical-review.md, Deep Technical Review - Open-Source OCR - Visual Document Understanding Models - Missed Details and Pipeline Implications.md]
created: 2026-08-27
updated: 2026-08-27
---

# Zhipu AI

Also Z.ai. Publisher of [[GLM-OCR]] (Feb 2026, arXiv 2603.10910) and the [[glmocr SDK]] that ships with it.

The interesting thing about Zhipu's contribution here is that its **most durable artifact is not the model**. GLM-OCR was disqualified as a default by an 8-language core that excludes Hungarian — but the SDK around it (layout integration, region fan-out, reading-order assembly, formatting, env-var config) survived the model's rejection and became the fixed point of the architecture. See [[Pipeline as Platform, Model as Config]].

Two further reusable contributions from the paper:

- The **GRPO reward table** (NED for text, CDM for formulas, TEDS for tables, field-F1 for KIE, plus global repetition and malformed-structure penalties) is a copyable objective design for a format-native fine-tune.
- The **MTP head reused as a speculative draft** at serving time — see [[Multi-Token Prediction and Speculative Decoding]].

Licensing is clean: MIT weights, with the bundled Apache-2.0 [[PP-DocLayout-V3]] from [[Baidu]]. API key env var is `ZHIPU_API_KEY`; a hosted MaaS API and an agent "Skill" mode exist alongside self-hosting.

## Related

[[GLM-OCR]] · [[glmocr SDK]] · [[Baidu]]
