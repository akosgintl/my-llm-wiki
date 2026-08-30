---
aliases: ["Cost per Page Model"]
tags: [synthesis, economics, serving, throughput, decision]
sources: [ocr-serving-recipes-runpod-v2.md, ocr-vdu-complete-study.md, Deep Technical Review - Open-Source OCR - Visual Document Understanding Models - Missed Details and Pipeline Implications.md, tiered-pdf-pipeline-architecture.md, ocr-api-call-strategy-images-concurrency.md]
created: 2026-08-27
updated: 2026-08-27
---

# Cost per Page Model

Throughput figures are scattered across ten entity pages in three incompatible units — pages/s, images/s, pages/day — on two different GPUs. **Nobody can compare them by eye, so nobody does, and tiering decisions get made on vibes.** This page converts everything to one number: **dollars per 1,000 pages.**

## The formula

```
cost per 1k pages  =  (GPU $/hr × 1000) / (pages_per_second × 3600)
```

Everything below assumes **$2.00/hr** for a GPU — the midpoint of the €0.50–2.50/hr H100 band recorded for Scaleway in [[EU Sovereign GPU Hosting]]. Substitute your own rate; the *ratios* between rows are what matter and they do not move.

**Two caveats, stated once:** several source figures were measured on A100 (cheaper than the $2.00 assumed here, so those rows are **overstated**), and none of the published figures share a batch size or backend. Treat this as order-of-magnitude, and re-measure on your own corpus before committing — the same discipline as [[Golden Set and Eval Harness]].

## Recognition tier

| Model | Throughput | GPU measured | **$ / 1k pages** |
|---|---|---|---|
| [[LightOnOCR]] (peak figure) | 42.8 pages/s | H100 | **$0.013** |
| [[LightOnOCR]] (sustained) | 5.71 pages/s | H100 | **$0.097** |
| [[DeepSeek-OCR]] | ~2.31 pages/s (200k/day) | A100-40G | **$0.240** |
| [[MinerU]] | 2.12 pages/s | A100 | **$0.262** |
| [[PaddleOCR-VL]] (FastDeploy) | ~2.0 pages/s | A100 | **$0.278** |
| [[GLM-OCR]] | 1.86 pages/s | — | **$0.299** |
| [[olmOCR]] | 1.78 pages/s | H100-class | **$0.312** |
| [[PaddleOCR-VL]] ([[vLLM]]) | ~1.38 pages/s | A100 | **$0.403** |
| [[dots.ocr]] | ~0.35 pages/s | A100 | **$1.587** |
| *Mistral OCR 4 (hosted API)* | — | — | *~$4.00* |

## What the conversion exposes

**1. The sub-cent claim rests on the peak figure, not the sustained one.** [[LightOnOCR]]'s recorded "under $0.01 per 1k pages" is only reachable at the **42.8 pages/s peak** — at the 5.71 pages/s sustained figure the same GPU rate gives ~$0.10, a **7.5× difference**. Both numbers appear in the corpus without the distinction being drawn. Use 5.71 for capacity planning; treat 42.8 as a best-case ceiling.

**2. The accuracy leader costs 16× the throughput leader.** [[dots.ocr]] at $1.59/1k against LightOnOCR's $0.097/1k sustained. This is the real shape of the trade-off in the [[Hungarian Model Decision Matrix]]: the strongest independent scores carry a **16× cost multiple**, which is exactly the size of gap that [[Tiered Page Routing]] exists to close.

**3. Self-hosting beats the hosted API by 2.5×–40×** — but only after fixed costs. Mistral OCR 4 at ~$4/1k is 13–40× the self-hosted band. The crossover depends entirely on volume, and that number is not in the wiki because it depends on your corpus size. See [[Open Inputs and Corpus Profile]].

## Layout tier — why it is nearly free

From [[Layout Stage Economics]], converted the same way:

| Placement | Rate | Throughput | **$ / 1k pages** |
|---|---|---|---|
| Shared T4/L4 micro-service (TensorRT) | ~$0.50/hr | 25–60 pages/s | **$0.002–0.006** |
| CPU sidecar (ONNX/OpenVINO INT8, 8 vCPU) | ~$0.35–0.40/hr | 3–6 pages/s | **$0.017–0.035** |

**The layout stage costs 17–130× less than the cheapest recognition tier.** The wiki's recorded "10–50× cheaper" is *conservative*. The operational consequence stands unchanged: one ~$0.50/hr T4 feeds a seven-replica H100 farm, so layout placement is a rounding error and should never be optimised before recognition.

## Why tiering is the dominant lever

Tier 1 ([[liteparse]], CPU, milliseconds/page) is effectively free next to any GPU tier. So the blended cost is set almost entirely by **what fraction of pages escalate to Tier 3**:

| Share of pages hitting Tier 3 | Blended cost / 1k pages (at $0.30 recognition) |
|---|---|
| 100% | $0.300 |
| 50% | $0.150 |
| 20% | $0.060 |
| 5% | $0.015 |

**Moving the escalation rate from 100% to 20% saves 5×** — more than any model swap on the recognition table above, and it costs no accuracy if the router is right. This is the concrete economic argument behind [[Tiered Page Routing]], and it is why the router's five signals deserve more tuning attention than the model choice does.

It is also the argument for [[Pipeline as Platform, Model as Config]] in cash terms: the escalation rate is a **pipeline** property, and it dominates the **model** property.

## The number this page cannot give you

Total cost = cost per 1k pages × your volume. **Volume is not in the wiki** — it is the first of the blocking inputs listed in [[Open Inputs and Corpus Profile]]. Until it exists, every figure here is a rate without a quantity.

## Related

[[Layout Stage Economics]] · [[Tiered Page Routing]] · [[Hungarian Model Decision Matrix]] · [[EU Sovereign GPU Hosting]] · [[MIG and GPU Sharing]] · [[vLLM Continuous Batching and Concurrency Sizing]] · [[Open Inputs and Corpus Profile]]
