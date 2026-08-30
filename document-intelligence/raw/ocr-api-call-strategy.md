# API Call Strategy: Images per Prompt, Concurrency, Parallelism & Load Balancing
*Companion to the vLLM recipes (Aug 2026). Covers: (1) images-per-request limits per model, (2) client concurrency sizing, (3) the full parallelism stack from request to silicon, (4) load balancing in front of MIG slices.*

---

## 1. Images per prompt — hard truths per model

| Model | Images / request | Why | Multi-page document strategy |
|---|---|---|---|
| GLM-OCR | **1** (a region crop) | Two-stage design: PP-DocLayout-V3 crops regions, model OCRs each crop | Pipeline fans out N regions × M pages as N×M parallel single-image requests, reassembles |
| PaddleOCR-VL-1.6 | **1** (a region crop) | Same two-stage pattern | Same region fan-out; cross-page table merge happens in post-processing |
| DeepSeek-OCR / 2 | **1** (a full page) | End-to-end page model; no multi-image prompt | Page fan-out: 1 request per page, stitch markdown in order |
| **Unlimited-OCR** | **Multi-image: ~3–40 pages** in ONE request (base mode, 1024px, `infer_multi` / multiple `image_url` parts) | R-SWA flat KV cache — this is its entire reason to exist | The document IS the request. Practical ceiling is the **32,768-token max_length** (prompt + output): dense pages emit ~600–900 output tokens each → ~30–40 text pages, fewer for dense tables. Gundam mode = single image only |
| dots.ocr | **1** (a full page) | End-to-end page model | Page fan-out + stitch |
| MinerU | N/A — accepts whole PDFs at its own API | Platform handles paging internally | Send the file, not pages |

Two consequences worth internalizing:

**a) Everything except Unlimited-OCR is a single-image-per-request system.** Your event-driven gateway's job is fan-out/fan-in: rasterize → 1 request per page (or let GLM/Paddle SDKs do region fan-out) → ordered reassembly. This is good news — single-image requests are the ideal shape for vLLM continuous batching.

**b) For Unlimited-OCR, budget tokens, not pages.** Rule of thumb before submitting: `estimated_output ≈ pages × 800`; if `pages × 800 + pages × ~300 vision tokens > 30K`, split the document into chunks of ~25–30 pages with 1-page overlap and merge. Don't send a 60-page contract as one request and hope.

---

## 2. Concurrent calls — sizing the client against `--max-num-seqs`

vLLM continuous batching means the server itself is the batcher: you never build image batches client-side; you keep enough single requests in flight to keep the batch full.

**The sizing rule:** per replica, keep `in-flight ≈ 1.0–1.5 × max_num_seqs`. Below 1.0× the GPU starves between completions; far above it requests sit in vLLM's internal queue inflating p99 latency and risking client timeouts. Enforce with an async semaphore per replica (or per LB pool: `Σ max_num_seqs × 1.25`).

| Model (config from recipes) | max_num_seqs | Client in-flight per replica | Notes |
|---|---|---|---|
| GLM-OCR (24 GB min / 1g.10gb) | 32 / 64 | 40 / 80 | Region requests are short — high turnover, feed aggressively |
| GLM-OCR (2g.20gb) | 128 | ~160 | |
| PaddleOCR-VL (min / 2g.20gb) | 32 / 96 | 40 / 120 | Cap also by CPU layout-stage throughput upstream |
| DeepSeek-OCR (min / full H100) | 16 / 256 | 20 / 300 | Pages emit 1–4K tokens — longer residency per seq |
| Unlimited-OCR (min / full H100) | 8 / 32 | **8 / 32 — never overdrive** | Each seq is a whole document holding a slot for minutes; queue at the gateway, not in vLLM |
| dots.ocr (min / 3g.40gb) | 4 / 24 | 4–5 / 28 | Memory balloons — treat max_num_seqs as a hard ceiling |
| MinerU | engine-managed | send docs, let router meter | |

```python
# Gateway pattern (per model pool)
import asyncio, httpx

SEM = asyncio.Semaphore(int(TOTAL_MAX_NUM_SEQS * 1.25))

async def ocr_page(client: httpx.AsyncClient, payload: dict):
    async with SEM:
        for attempt in range(4):                       # retry loop
            try:
                r = await client.post(f"{LB_URL}/v1/chat/completions",
                                      json=payload, timeout=120)
                if r.status_code in (429, 503):
                    raise TransientError()
                return r.json()
            except (TransientError, httpx.TimeoutException):
                await asyncio.sleep((2 ** attempt) + random.random())  # jittered backoff
```

Timeout guidance: 60–120 s for page models; 10–30 **minutes** for Unlimited-OCR long documents (and use streaming so the connection proves liveness). Retries: idempotent by design — an OCR request can always be resent; add a loop-detector on the response (n-gram repetition, output-length ceiling) and count a looped output as a retryable failure.

---

## 3. The full parallelism stack (what applies, what doesn't)

From request to silicon, you have six layers. For 0.9–3B OCR models the leverage is at the top and bottom; the middle (tensor/pipeline parallelism) is explicitly NOT for you.

**L1 — Document/page fan-out (your gateway).** Queue-per-model (Service Bus/Redis Streams), workers pull, rasterize on CPU pods, fan out pages/regions, fan in ordered results. Priority lanes: separate queues (or vLLM `--scheduling-policy priority` with per-request `priority`) so interactive KIE isn't stuck behind a 5,000-page batch job.

**L2 — Request concurrency (semaphores above).** Also: vLLM's API server itself can bottleneck at very high RPS with tiny requests (GLM-OCR region traffic) — recent vLLM supports `--api-server-count N` to run multiple frontend processes over one engine; use it on the GLM-OCR replicas before adding GPUs.

**L3 — Continuous batching + chunked prefill (inside vLLM, free).** Continuous batching is automatic. Keep chunked prefill enabled (default in modern vLLM): image-heavy prefills would otherwise stall in-flight decodes — exactly your workload shape. Keep `--no-enable-prefix-caching` everywhere: OCR requests share no prefix; the cache only costs memory.

**L4 — Speculative decoding.** GLM-OCR's MTP (`--speculative-config '{"method":"mtp",...}'`) is the one model where this applies — keep it. Don't bolt draft-model speculation onto the others; OCR output is cheap-per-token already on MoE decoders.

**L5 — Data parallelism = replicas.** The workhorse. Two flavors:
- **External DP (recommended):** N independent vLLM processes (one per MIG slice / GPU) behind a load balancer. Simple, failure-isolated, autoscales per pool.
- **vLLM internal DP** (`--data-parallel-size N`): one launch command spreading engine replicas across GPUs with a built-in coordinator. Convenient on a multi-GPU node, but you lose per-replica health/scaling granularity — for an event-driven K8s setup, external DP wins.
- **Tensor parallelism (`--tensor-parallel-size`): don't.** TP pays off when a model doesn't fit one device; every model here fits in 10–40 GB. TP overhead would make these models slower. Same for pipeline parallelism.

**L6 — GPU sharing (when replicas share silicon).** Three mechanisms, one clear winner for prod:

| Mechanism | Isolation | Memory protection | Latency behavior | Verdict |
|---|---|---|---|---|
| **MIG** | Hardware (SM + memory slices) | Yes — hard | Deterministic per slice | ✅ Production default (H100/A100) |
| MPS | Process co-scheduling on shared SMs | **No** — one OOM/crash can take down neighbors | Good utilization, noisy-neighbor risk | Only for squeezing GLM-OCR replicas onto non-MIG GPUs (A10/L4/4090 can't MIG) |
| Time-slicing | None (context switch) | No | Worst p99 | ❌ Avoid for inference |

On the 24 GB common-minimum fleet (no MIG support on A10/L4/4090): run ONE vLLM process per card. Two processes at `--gpu-memory-utilization 0.45` each is possible but couples their failure domains for little gain on 3B models; only GLM-OCR is small enough to consider it, and MPS + 2×0.9B on an L4 is the only defensible combo.

---

## 4. Load balancer in front of MIG slices

**The problem with the obvious answer:** a plain Kubernetes Service (ClusterIP) balances at TCP-connection level, and HTTP keep-alive means one client connection pins to one slice — you'll see one MIG slice at 100% and six idle. You need L7, per-request, and ideally load-aware balancing.

**Ranked recommendations:**

**Tier 1 — Kubernetes Gateway API Inference Extension (GIE)** — the K8s-native, purpose-built answer (and AKS supports Gateway API via managed gateways). Its endpoint-picker routes each request on live vLLM signals — queue depth (`num_requests_waiting`) and KV-cache utilization scraped from `/metrics` — which is exactly the right signal for OCR: since there are no shared prefixes, "least-loaded replica" is the *optimal* policy (prefix-aware routing, the other big GIE feature, buys you nothing here — skip it). Define one `InferencePool` per model (glm-ocr pool = 7 MIG-slice pods, dots-ocr pool = 2, …), route on model name.

**Tier 2 — LiteLLM proxy** — fastest to stand up, OpenAI-compatible, gives you per-model routing (`model: glm-ocr` → GLM pool), least-busy/lowest-latency balancing strategies, health checks, automatic retries/fallbacks (e.g. dots.ocr overflow → GLM-OCR fallback), budgets and rate limits per caller. Runs as a simple Deployment; config is a YAML listing your replica URLs. Ideal while the fleet is < ~20 replicas or on RunPod where you don't control the ingress layer.

**Tier 3 — Envoy/Istio `LEAST_REQUEST` or NGINX `least_conn`** — if you already run a mesh/ingress. Least-outstanding-requests is a good proxy for vLLM load when request costs are homogeneous (true for page-model pools; NOT true if you mix page and long-doc traffic in one pool — don't). Add active health checks against vLLM `/health` and outlier ejection so a wedged slice is evicted.

**Tier 4 — vLLM production-stack / llm-d routers** — bring KV-aware and prefix-aware scheduling. Overkill for OCR (no prefix reuse); revisit only if you later co-host chat/RAG models on the same fleet.

**Topology that ties it together (AKS):**
```
Service Bus / Redis queues (per model, per priority)
        │  KEDA scales pools on queue depth
        ▼
CPU gateway pods ── rasterize @ fixed DPI, fan-out, semaphores, retries, loop-detector
        ▼
Gateway API + Inference Extension (or LiteLLM)
   ├── InferencePool: glm-ocr      → 7 × 1g.10gb MIG pods   (H100 #1)
   ├── InferencePool: paddleocr-vl → 3 × 2g.20gb MIG pods   (H100 #2)
   ├── InferencePool: dots-ocr     → 2 × 3g.40gb MIG pods   (H100 #3, mixed w/ GLM 1g slices)
   ├── InferencePool: deepseek-ocr → full-GPU pods (batch, spot-friendly)
   └── InferencePool: unlimited-ocr→ full-GPU H100 pods — SEPARATE pool, never mixed
        ▼
fan-in / reassembly → downstream extraction
```
Per-pool rules that matter more than the LB choice:
- **Never mix Unlimited-OCR into a page-model pool.** A 20-minute document request behind the same LB as 2-second region requests wrecks every latency-based balancing signal. Separate pool, separate queue, `in-flight = max_num_seqs`, period.
- **Readiness ≠ liveness for vLLM:** readiness = `/health` after model load (gate with a generous `startupProbe`, CUDA-graph capture takes minutes); liveness should also fail on sustained `num_requests_waiting` growth (wedged engine detector).
- Expose `/metrics` to Prometheus; alert on `num_requests_waiting > max_num_seqs` sustained >60 s per replica — that's your "scale out or shed load" signal, and with KEDA it can literally be the scaler.

**On RunPod (no MIG, no ingress control):** replicas are separate pods/workers; put LiteLLM (or your gateway with client-side least-busy over the replica URL list) in front. RunPod Serverless endpoints come with their own queue+autoscaler — in that mode drop your own LB for that pool and let `scalerType: QUEUE_DELAY` do L5 for you; keep the semaphore/timeout/retry layer regardless.

---

## TL;DR decision card
1. Single image per request everywhere except Unlimited-OCR (3–40 pages/request, 32K token budget — chunk at ~25–30 pages).
2. Client in-flight = 1.25 × Σ max_num_seqs per pool, async semaphore, jittered retries, loop-detector-as-retry.
3. Parallelism = fan-out (L1) + continuous batching (L3) + replicas (L5) + MIG (L6). Skip TP/PP entirely; MTP only on GLM-OCR; MPS only for 0.9B on non-MIG cards; never time-slice.
4. LB = Gateway API Inference Extension on AKS (least-loaded via vLLM metrics; skip prefix-aware — useless for OCR), LiteLLM as the pragmatic alternative; per-model pools; Unlimited-OCR quarantined in its own pool.
