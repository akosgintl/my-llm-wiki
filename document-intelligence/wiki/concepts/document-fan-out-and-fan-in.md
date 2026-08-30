---
aliases: ["Document Fan-Out and Fan-In"]
tags: [architecture, serving, latency, pipeline]
sources: [ocr-api-call-strategy.md, ocr-vdu-complete-study.md, ocr-rasterize-encoding-bottlenecks.md]
created: 2026-08-27
updated: 2026-08-27
---

# Document Fan-Out and Fan-In

**Everything except [[Baidu Unlimited-OCR]] is a single-image-per-request system.** The gateway's whole job is therefore: rasterize → one request per page or region → ordered reassembly.

That is good news — single-image requests are the ideal shape for [[vLLM]] continuous batching.

## Images per request, per model

| Model | Images/request | Multi-page strategy |
|---|---|---|
| [[GLM-OCR]] | 1 (a **region crop**) | N regions × M pages as N×M parallel single-image requests |
| [[PaddleOCR-VL]] | 1 (a region crop) | same region fan-out; cross-page table merge in post-processing |
| [[DeepSeek-OCR]] | 1 (a **full page**) | page fan-out, stitch markdown in order |
| [[dots.ocr]] | 1 (a full page) | page fan-out + stitch |
| **[[Baidu Unlimited-OCR]]** | **~3–40 pages in ONE request** | the document *is* the request |
| [[MinerU]] | N/A — takes whole PDFs | send the file, not pages |

**For Unlimited-OCR, budget tokens, not pages.** Its 32K `max_length` covers prompt + reference + output together: `pages × ~800 output + ~300 vision tokens/page + prompt ≤ ~30K` → chunk at ~25–30 pages with 1-page overlap.

## The 200-page latency playbook

Latency comes from making the document **one continuous-batching wave**: `total ≈ ceil(200 / Σ max_num_seqs) × wave_time`.

1. **Text-layer check first** — born-digital pages need no GPU at all. OCR only the image pages.
2. **Parallel rasterization** at target resolution: ~1–2 s for 200 pages on 16 cores.
3. **Fire all 200 at once.** At a full-H100 DeepSeek-OCR replica (seqs 256): ~1–2 min single-card *estimate* (unvalidated — benchmark before promising an SLA). On a 24 GB fleet you need ~12–13 replicas for one wave; fewer replicas multiply latency linearly.
4. **Stream fan-in by page index** — downstream starts on page 1 while page 200 is still decoding.
5. **Hedge stragglers** — set a per-page deadline at the observed p95, re-issue laggards to another replica, first result wins. Worth **20–30%** off total.
6. **Cross-page tables**: fast fan-out first, detect split tables in post, then send only those 2–3 page spans to Unlimited-OCR as small multi-image requests.

**Do not use Unlimited-OCR for lowest latency.** Its one-shot design is for *fidelity*: 200 pages ≈ 160K output tokens ⇒ ~7 sequential chunks, and a one-sequence model cannot parallelize within a document.

## Gateway mechanics

```python
SEM = asyncio.Semaphore(int(TOTAL_MAX_NUM_SEQS * 1.25))

async def ocr_page(client, payload):
    async with SEM:
        for attempt in range(4):
            try:
                r = await client.post(f"{LB_URL}/v1/chat/completions", json=payload, timeout=120)
                if r.status_code in (429, 503):
                    raise TransientError()
                return r.json()
            except (TransientError, httpx.TimeoutException):
                await asyncio.sleep((2 ** attempt) + random.random())   # jittered backoff
```

- **Timeouts**: 60–120 s for page models; **10–30 minutes** for Unlimited-OCR long documents, with streaming so the connection proves liveness.
- **Retries are safe by design** — an OCR request is idempotent and can always be re-sent. Add a loop detector on the response and **count a looped output as a retryable failure**. See [[Repetition Loops in VLM OCR]].
- **Queue documents, not pages.** Azure Service Bus adds ~20–100 ms per message — fine for job intake, wrong for 200 per-page messages on the hot path. Enqueue the document, fan out pages in-process (or Redis if you must distribute).
- **Priority lanes** — separate queues, or vLLM `--scheduling-policy priority` with per-request priority, so interactive KIE is not stuck behind a 5,000-page batch.

## Failure semantics nobody specs and everybody ships wrong

Partial delivery + a **per-page status manifest** + idempotency keys + a dead-letter queue that carries the page image. Decide this before building more.

## Related

[[vLLM Continuous Batching and Concurrency Sizing]] · [[Parallelism Stack for OCR Serving]] · [[Load Balancing Inference Pools]] · [[Base64 and Image Transport]]
