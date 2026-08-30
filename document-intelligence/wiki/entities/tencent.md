---
aliases: ["Tencent"]
tags: [organization]
sources: [Deep Technical Review - Open-Source OCR - Visual Document Understanding Models - Missed Details and Pipeline Implications.md, ocr-arxiv-github-technical-review.md, Open-Source OCR per Visual Document Understanding Models for Self-Hosted AKS Pipelines - A Comparative Investigation (August 2026).md]
created: 2026-08-27
updated: 2026-08-27
---

# Tencent

Publisher of [[HunyuanOCR]] (v1.0 arXiv 2511.19575, v1.5 arXiv 2607.04884) and of **[[CHAOS-Bench]]**, the character-level faithfulness benchmark introduced alongside it.

Their contribution is a genuinely different optimization target. Where most of the field optimizes for benchmark accuracy on clean print, Tencent optimized for **not inventing text when the pixels are ambiguous** — and then published the benchmark that measures it. HunyuanOCR-1.5 scores 14.15 on CHAOS-Bench against 3–6.3 for every peer.

That combination of strong multilingual coverage (top open model on MORE at 92.42) and faithfulness makes it the natural "evidence-preserving" arm of a confidence architecture.

Two practical cautions: the license is **NOASSERTION** and needs counsel review before commercial use, and the repo notes a **[[vLLM]]-versus-transformers accuracy gap under repair** — pin and A/B the engines before trusting bake-off numbers.

## Related

[[HunyuanOCR]] · [[CHAOS-Bench]] · [[Linguistic Crutch and Faithfulness]]
