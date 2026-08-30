---
aliases: ["OCR Arena"]
tags: [benchmarks, evaluation, document-parsing]
sources: [Open-Source OCR per Visual Document Understanding Models for Self-Hosted AKS Pipelines - A Comparative Investigation (August 2026).md, ocr-vdu-complete-study.md]
created: 2026-08-27
updated: 2026-08-27
---

# OCR Arena

`ocrarena.ai`, built by Extend.ai — a **blind community ELO leaderboard** for document OCR. Its value is that it is neither vendor-run nor metric-gameable: humans compare outputs without knowing which model produced them.

## What it says

| Model | Rank | ELO |
|---|---|---|
| [[dots.ocr]] | #12 | 1442 |
| [[olmOCR]] 2 | #19 | 1382 |
| [[GLM-OCR]] | #24 | 1347 |

Head-to-head: dots.ocr beats GLM-OCR with an **88.9% win rate across 9 matchups**; olmOCR-2 beats it with a **92.3% win rate across 13 matchups**. [[DeepSeek-OCR]] v1 historically sat last.

## Why it matters

GLM-OCR tops [[OmniDocBench]] at 94.62 and sits at #24 here. That gap — vendor-reported metric leadership versus blind human preference — is the clearest single piece of evidence for [[Benchmark Saturation]], and the reason a [[Golden Set and Eval Harness]] on your own corpus is non-negotiable.

## Related

[[OmniDocBench]] · [[Multi-Script OCR Benchmarks]] · [[Benchmark Saturation]]
