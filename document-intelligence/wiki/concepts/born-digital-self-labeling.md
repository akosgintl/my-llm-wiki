---
aliases: ["Born-Digital Self-Labeling"]
tags: [fine-tuning, data, methodology, hungarian]
sources: [ocr-language-models-deployment-critical-pass-2.md, ocr-vdu-complete-study.md]
created: 2026-08-27
updated: 2026-08-27
---

# Born-Digital Self-Labeling

The two cheats that make a 10–30K-pair training set cost roughly zero annotation hours — and the reason **you should build this pipeline regardless of the fine-tune decision**, because it also feeds the [[Golden Set and Eval Harness]].

## Cheat 1: born-digital PDFs are free labels

A born-digital PDF already contains its own ground truth in the text layer. So:

1. render the page to an image at target resolution,
2. extract the text layer,
3. pair them.

A few thousand born-digital Hungarian documents yields the 10–30K training pairs with **zero annotation cost**. The same text-layer check is also the first step of the 200-page latency playbook (born-digital pages need no GPU at all) and the router signal in [[Tiered Page Routing]] — one mechanism, three uses.

## Cheat 2: synthetic rendering scales the tail

Hungarian corpora — Wikipedia-hu, OSCAR-hu, the Hungarian National Corpus, your own domain text — rendered to pages with varied fonts and layouts, then degraded with **augraphy** (noise, blur, skew) to imitate the scanned fraction.

**100K pages ≈ one afternoon of compute.** Mix roughly **80% synthetic / 20% real**.

## Why this matters more than model choice

[[MinerU]]2.5-Pro gained +5 points on OmniDocBench v1.6 with an **identical architecture**, purely through data engineering. That is the strongest published evidence that a data plan outperforms architecture chasing — and this pipeline is the data plan.

## Caveat

Born-digital text layers are ground truth only when they are *correct*. Older Hungarian PDFs frequently carry broken Latin-2 / CP-1250 text layers with `õ` standing in for `ő` — the exact character class you are trying to teach. **Filter for mojibake before using a text layer as a label**, or you will train the model to reproduce the bug. See [[Hungarian OCR and the Double Acute Test]].

## Related

[[LoRA Fine-Tuning for OCR]] · [[Golden Set and Eval Harness]] · [[Tiered Page Routing]] · [[MinerU]]
