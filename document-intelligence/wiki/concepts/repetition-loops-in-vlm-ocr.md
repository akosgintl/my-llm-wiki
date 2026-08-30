---
aliases: ["Repetition Loops in VLM OCR"]
tags: [failure-modes, ocr, serving, reliability]
sources: [Open-Source OCR per Visual Document Understanding Models for Self-Hosted AKS Pipelines - A Comparative Investigation (August 2026).md, ocr-serving-recipes-runpod-v2.md, ocr-api-call-strategy.md, ocr-vdu-complete-study.md, ocr-arxiv-github-technical-review.md, Deep Technical Review - Open-Source OCR - Visual Document Understanding Models - Missed Details and Pipeline Implications.md]
created: 2026-08-27
updated: 2026-08-27
---

# Repetition Loops in VLM OCR

**The #1 production failure mode across all VLM-based OCR.** LlamaIndex traced LlamaParse outages to exactly this. Any pipeline that does not detect it will silently emit garbage documents.

## The measured scale

| Source | Rate |
|---|---|
| [[DeepSeek-OCR]] on 600 British Library historical newspaper clippings (Jim Clifford, University of Saskatchewan, Issue #151) | **9.2% catastrophic failure cohort** despite guardrails |
| DeepSeek's own OCR-2 measurement, user-log images | 6.25% → 4.17% |
| DeepSeek's own OCR-2 measurement, PDF production | 3.69% → 2.88% |
| [[Baidu Unlimited-OCR]] anti-loop metric (Distinct-35) | 96.90% |

Note the vendors measure repetition rate in production *because ground truth is unavailable there* — it is one of the few quality signals you can compute without labels.

## Triggers

- **Long, dense, handwritten, non-English or vertical text** (DeepSeek lineage).
- **Runs of continuous special characters** — ellipses and underscores. [[dots.ocr]] documents this explicitly, and it means **dotted table-of-contents leader lines and underscore form blanks**, i.e. exactly the furniture on Hungarian invoices and forms. Pre-strip or regex-guard them.
- Fewer visual tokens → more reliance on language priors → more looping and hallucination. See [[Linguistic Crutch and Faithfulness]].

## Four control patterns

**1. n-gram logits processors — mandatory, not optional.**

```bash
# DeepSeek-OCR
--logits-processors vllm.model_executor.models.deepseek_ocr:NGramPerReqLogitsProcessor
# per request: {"ngram_size": 30, "window_size": 90}

# Unlimited-OCR
--logits_processors vllm.model_executor.models.unlimited_ocr:NGramPerReqLogitsProcessor
# per request: {"ngram_size": 35, "window_size": 128}  (1024 for multi-page)
```

Also requires `skip_special_tokens: false`, `temperature=0.0`.

**2. [[R-SWA Reference Sliding Window Attention]]** — [[Baidu Unlimited-OCR]]'s architectural answer: bounded output-token attention makes long-horizon looping structurally harder.

**3. Dynamic per-element suppression** — [[MinerU]]'s frequency and presence penalties applied per layout element at decode, tuned so they do **not** kill legitimately repetitive structures like tables and equations. The conceptually best of the three.

**4. Trained-in penalties** — [[GLM-OCR]]'s GRPO reward table includes an explicit global repetition-ratio penalty and a malformed-structure penalty. The anti-loop behavior is trained, not emergent. [[LightOnOCR]] includes 3% blank-page examples for the same reason.

## The pipeline-level requirement

Regardless of model, add a **loop detector as a retry trigger**:

- n-gram repetition detection on the response
- output-length ceilings
- blank-page handling

An OCR request is idempotent by design, so it can always be re-sent. **Count a looped output as a retryable failure** — and budget for it: ~9% catastrophic on hard scans means one page in eleven costs you a second inference. Straggler pages also define your p100, so hedged re-issue at the observed p95 deadline is worth 20–30% of total latency. See [[Document Fan-Out and Fan-In]].

## Related

[[Linguistic Crutch and Faithfulness]] · [[Confidence Engineering]] · [[Document Fan-Out and Fan-In]] · [[DeepSeek-OCR]] · [[dots.ocr]]
