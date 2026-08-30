# Self-Hosted OCR / Visual Document Understanding on AKS — The Complete Study
*Consolidated master document (August 2026). Merges: the model-landscape research, vLLM serving recipes + RunPod API v2, API-call & concurrency strategy, the 200-page latency playbook, rasterization/encoding/bottleneck analysis, the deployment & EU-sovereignty map, the Hungarian-language strategy and fine-tuning plan, layout-stage economics, the glmocr-SDK model-swap architecture, the Qwen family map, downstream LLM roles, and confidence engineering. Figures marked ~ are engineering ballparks; benchmark and pricing claims are Feb–Aug 2026 snapshots.*

---

## 0. Executive summary — the two rules and the decision card

**Rule 0: the golden set is the most valuable artifact in the project.** 50–100 labeled pages from the real corpus + an eval harness (CER, word accuracy, **diacritic-specific error rate**, TableTEDS) converts every decision — model, fine-tune, cloud, confidence threshold, version upgrades — from vibes to measurement. One engineer-week. Build it before anything else.

**Rule 0.5: pipeline is the fixed point, the model is a config value.** The glmocr SDK (layout, fan-out, assembly, formatting) is where durable value lives; the recognition model behind its OpenAI-compatible HTTP contract is swappable via environment variables. This makes the 2-month model-release treadmill irrelevant.

**Decision card:**
1. Corpus is 80% Hungarian → GLM-OCR *the model* is out; **Qwen3.5-4B behind the glmocr SDK is the front-runner** (201 languages, native multimodal, right size, day-1 fine-tune tooling), with PaddleOCR-VL-1.6 and dots.ocr as specialist arms.
2. Serving: **vLLM only**; common practical minimum = **24 GB, Ampere+ (CC ≥ 8.0)** (RTX 4090 on RunPod / A10 on Azure / L4 elsewhere); H100 via MIG packing per model.
3. Fan-out architecture: single-image requests everywhere except Unlimited-OCR; client in-flight ≈ 1.25 × Σ max_num_seqs; per-model LB pools (Gateway API Inference Extension or LiteLLM); Unlimited-OCR quarantined in its own pool.
4. Rasterize at model-native resolution (long edge 1024–1600 px), JPEG q90 turbo, permissive-license tools (pypdfium2 / pdftoppm); base64-encode CPU is a non-issue — the real costs are JSON of megabyte strings, vLLM frontend CPU, and WAN transfer.
5. Deployment posture: AKS control plane · EU-sovereign GPU burst (Scaleway/OVHcloud — CLOUD Act follows *incorporation*, not geography) · RunPod/Modal for non-sensitive work only.
6. Fine-tune only if the probe demands it: Unsloth LoRA, vision encoder frozen, 10–30K pairs from born-digital self-labeling + synthetic rendering, **format-native targets** (one run buys language + output contract).
7. Confidence is engineered, not emitted: layout scores + vLLM logprobs + value-in-OCR grounding + Hungarian checksums, calibrated on the golden set; threshold τ routes to human review.

---

## 1. Model landscape (Aug 2026)

Document OCR has converged on compact (0.9–3B) vision-language models emitting structured Markdown/JSON. The standard benchmark, OmniDocBench v1.5/v1.6, is **saturated and Latin/CJK-print-biased** — top open models cluster 94–96 and independent evaluations reorder the field sharply.

**Vendor OmniDocBench (overall):** PaddleOCR-VL-1.6 96.33 · MinerU2.5-Pro 95.69 · GLM-OCR 94.62–95.2 · HunyuanOCR-1.5 94.74 · Unlimited-OCR 93.23/93.92 · DeepSeek-OCR-2 91.09 · dots.ocr ~88.4 · DeepSeek-OCR 87.01.

**Independent signal (the correction):** on multi-script socOCRbench, dots.ocr leads open models while DeepSeek-OCR/2 and MinerU fall near the bottom (DeepSeek-OCR v1 below Tesseract); on blind-voting OCR Arena, dots.ocr (#12) beats GLM-OCR (#24) with an 88.9% win rate and olmOCR-2 beats it at 92.3%; on GlotOCR (mid-resource scripts), GLM-OCR and DeepSeek-OCR-2 sit 40+ points below Gemini Flash-Lite. **Vendor deltas of 1–2 points mean nothing; your golden set is the only leaderboard that counts.**

**Per-model capsules:**
- **GLM-OCR (Zhipu, 0.9B, MIT):** CogViT + GLM-0.5B, MTP speculative decoding (~5 tok/step), two-stage with PP-DocLayout-V3, schema-JSON KIE, seals/receipts strength, 1.86 pages/s, ~3 GB VRAM. Weaknesses: 8-language core (no Hungarian), tiny/long-text, prompt rigidity. → SDK stays; model replaced.
- **PaddleOCR-VL-1.6 (Baidu, 0.9B, Apache-2.0):** NaViT + ERNIE-0.3B behind PP-DocLayoutV3; 109 languages; best-in-class on in-the-wild/skewed/vertical text; ~1.4–2 pages/s. Hard requirement CC ≥ 8.0; must run the *full two-stage pipeline* or accuracy claims don't reproduce.
- **DeepSeek-OCR / OCR-2 (3B MoE, ~570M active):** optical compression (97% at 10×), ~200k pages/day/A100; OCR-2 adds human-reading-order DeepEncoder V2. Production failure mode: repetition loops (~9% catastrophic on hard historical scans; vendor's own 2.9–4.2% rates) — the n-gram logits processor is mandatory. Best fine-tuning tooling (Unsloth).
- **Baidu Unlimited-OCR (3B MoE, MIT):** DeepSeek-OCR continue-trained (encoder frozen) with R-SWA constant-KV attention → one-shot parsing of dozens of pages in a 32K window; +6.2 over its baseline; ~35% faster at long outputs. Thin independent evidence; no bounding boxes.
- **dots.ocr (1.7B, MIT):** best open-model blind-vote accuracy, JSON layout + bboxes, strong low-resource languages; slow (~0.35 pages/s), VRAM-hungry at batch, skips text-dense regions on colorful layouts, loops on `…`/`___`.
- **MinerU 2.5/Pro (1.2B, custom Apache-based):** full ingestion *platform* (multi-format, backends, router, RAG integrations); strong Latin/CJK print, collapses on non-Latin scripts.
- **HunyuanOCR-1.5 (~1B):** top open multilingual (MORE 92.42), most faithful reproduction (least "corrects" the source).
- Also: olmOCR-2 (7B, English RAG-corpus), LightOnOCR-2-1B (European-language focus — Hungarian-relevant), Granite-Docling-258M, Nemotron-OCR (34.7 pages/s recognition-only), PP-OCRv6 (34M CTC — hallucination-free cross-checker), Mistral OCR 4 (managed API with confidence + boxes — comparison/fallback point only).

**Workload → model mapping:** single-page KIE/receipts → GLM-pipeline-class; multilingual/messy scans → PaddleOCR-VL; long multi-page contracts → Unlimited-OCR; bulk clean-PDF digitization → DeepSeek-OCR (+ retry budget); forms needing JSON+boxes → dots.ocr; multi-format ingestion platform → MinerU. **Day-1 discipline: ONE model + one escape hatch; add a second only when a measured failure class demands it.**

---

## 2. Serving: vLLM recipes, common minimum, H100 packing

**Common practical minimum — one GPU class serves everything: 24 GB, CC ≥ 8.0.** RunPod: RTX 4090. Azure/AKS: A10 (no L4 on Azure). Elsewhere: L4. Forced by PaddleOCR-VL's Ampere floor and dots.ocr's batch memory ballooning.

**Universal vLLM flags for OCR:** `--no-enable-prefix-caching` (no shared prefixes; the cache only costs memory), pre-rasterized fixed-DPI input, generous `startupProbe` (CUDA-graph capture 1–3 min), big vCPU on GPU pods (8–16 — the frontend does JSON+b64+image decode on CPU), `--api-server-count N` when small-request RPS is high.

**Per-model essentials (minimal → H100):**

| Model | Minimal (24 GB) | H100 strategy |
|---|---|---|
| GLM-OCR / Qwen3.5-4B slot | `vllm serve … --speculative-config '{"method":"mtp","num_speculative_tokens":3}' --max-model-len 8192 --gpu-memory-utilization 0.85 --max-num-seqs 32` (MTP is GLM/Qwen3.5-specific) | **MIG 7×1g.10gb replicas** (len 16K, seqs 64) — a 0.9–4B model can't saturate an H100; replicas are the win |
| PaddleOCR-VL-1.6 | `--trust-remote-code --max-model-len 16384 --gpu-memory-utilization 0.7 --max-num-seqs 32` + full pipeline in front | 3×2g.20gb (seqs 96); don't raise context — scale the CPU layout workers instead |
| DeepSeek-OCR/2 | `--logits-processors …deepseek_ocr:NGramPerReqLogitsProcessor --no-enable-prefix-caching --mm-processor-cache-gb 0 --max-num-seqs 16`; per-request `vllm_xargs {"ngram_size":30,"window_size":90}` | Full card, seqs 256 (bulk) or 2×3g.40gb; 20 GB slices leave thin KV headroom for 3B MoEs |
| Unlimited-OCR | dedicated image `vllm/vllm-openai:unlimited-ocr` + `…unlimited_ocr:NGramPerReqLogitsProcessor`, caches off, len 16K, seqs 8; `vllm_xargs {"ngram_size":35,"window_size":128}` (1024 for multi-page) | **Full GPU, no MIG** — len 32768, seqs 32; 40-page prefill wants the whole card; context IS the product |
| dots.ocr | `--trust-remote-code --max-num-seqs 4` (hard ceiling; memory balloons) | 2×3g.40gb only (seqs 24); converts capacity into batch, never speed |
| MinerU | `mineru-openai-server` (8 GB+; `pipeline` backend = 4 GB/CPU) | 3×2g.20gb + mineru-router |

MIG reference (H100-80GB): 1g.10gb×7 · 2g.20gb×3 · 3g.40gb×2 · 7g.80gb×1; one vLLM process per slice (`CUDA_VISIBLE_DEVICES=MIG-<uuid>`); AKS via GPU Operator `mig.strategy`. Mixed-fleet: 1×3g.40gb (dots or DeepSeek) + 4×1g.10gb (small-model farm) on one card; Unlimited-OCR always un-sliced. **RunPod exposes no MIG** — rent the smaller GPU per replica instead.

**RunPod REST API v2** (public beta): base `https://api.runpod.io/v2`, OpenAPI at `/v2/openapi.json`; `POST /v2/templates` (name, imageName, ports, env, dockerStartCmd, containerDiskInGb, isServerless) → one template per model → drives `POST /v2/pods` and `/v2/endpoints` (KEDA-like `scalerType: QUEUE_DELAY`, `flashboot`, `networkVolumeId` for the HF cache). GLM/Qwen/dots/Paddle also fit RunPod's prebuilt vLLM serverless worker; DeepSeek/Unlimited need custom images (logits-processor flags).

---

## 3. API-call strategy: images/prompt, concurrency, parallelism, load balancing

**Images per request:** GLM-OCR & PaddleOCR-VL = 1 *region crop* (pipelines fan out N regions × M pages); DeepSeek-OCR & dots.ocr = 1 *full page*; **Unlimited-OCR = 3–40 pages in one request** — but budget *tokens*, not pages: `pages × ~800 output + ~300 vision tokens/page ≤ ~30K` → chunk at ~25–30 pages with 1-page overlap. MinerU takes whole files.

**Concurrency sizing:** vLLM continuous batching is the batcher — never batch images client-side; keep `in-flight ≈ 1.0–1.5 × max_num_seqs` per replica via async semaphore (jittered exponential backoff on 429/503/timeout; loop-detected output = retryable failure). Exception: **Unlimited-OCR at exactly max_num_seqs** — each sequence is a whole document holding a slot for minutes; queue at the gateway. Timeouts: 60–120 s page models, 10–30 min long docs (stream for liveness). Watch httpx's silent default cap of 100 connections.

**The six-layer parallelism stack** (leverage at top and bottom; the middle is a trap):
L1 document/page fan-out (queues per model *and* per priority; enqueue documents, fan out pages in-process — per-page Service Bus messages cost 20–100 ms each) → L2 request semaphores (+`--api-server-count` for tiny-request RPS) → L3 continuous batching + chunked prefill (free) → L4 speculative decoding (MTP: GLM-OCR, Qwen3.5/3.6 only) → L5 data parallelism = external replicas behind an LB (preferred over vLLM internal DP for health/scaling granularity; **tensor/pipeline parallelism: never** — everything fits one device, TP would make these models slower) → L6 GPU sharing: **MIG** (hard isolation, production default) > MPS (no memory isolation; only for packing 0.9B replicas on non-MIG 24 GB cards) > time-slicing (never).

**Load balancing in front of MIG slices:** a plain K8s Service balances TCP connections and keep-alive pins clients to one slice — one hot, six idle. Use L7 per-request, load-aware:
1. **Gateway API Inference Extension** (AKS-native): routes on live vLLM queue depth / KV utilization — with no shared prefixes, least-loaded is provably optimal (skip its prefix-aware features).
2. **LiteLLM proxy**: pragmatic — per-model pools, least-busy, retries/fallbacks, budgets; the right choice on RunPod.
3. Envoy `LEAST_REQUEST` / NGINX `least_conn` + `/health` checks + outlier ejection, per homogeneous pool only.
Rules that outrank the LB choice: **never mix Unlimited-OCR into a page-model pool** (20-minute requests wreck every load signal); readiness = `/health` post-load, liveness also fails on sustained `num_requests_waiting > max_num_seqs` (>60 s) — which doubles as the KEDA scale signal.

---

## 4. The 200-page latency playbook

For lowest latency, **do not use Unlimited-OCR** — one-shot design is for fidelity: 200 pages ≈ 160K output tokens ⇒ ~7 sequential chunks; a one-sequence model can't parallelize within a document. Latency comes from making the document **one continuous-batching wave**: total ≈ `ceil(200 / Σ max_num_seqs) × wave_time`.

1. **Text-layer check first** — born-digital pages need no GPU at all; OCR only the image pages.
2. **Parallel rasterization** (16 CPU cores, target resolution): ~1–2 s for 200 pages.
3. **Fire all 200 at once** at a full-H100 DeepSeek-OCR replica (seqs 256): ~1–2 min single-card estimate (unvalidated — benchmark before promising an SLA). On the 24 GB fleet: need ~12–13 replicas for one wave; fewer replicas multiply latency linearly — the burst-rental case.
4. **Stream fan-in by page index** — downstream starts on page 1 while page 200 decodes.
5. **Hedge stragglers** — per-page deadline at observed p95, re-issue laggards to another replica, first result wins (~20–30% off total).
6. Cross-page tables that matter: fast fan-out first, detect split tables in post, send only those 2–3 page spans to Unlimited-OCR as small multi-image requests.

---

## 5. Rasterization, encoding & the real bottlenecks

**The #1 self-inflicted wound: resolution.** Rendering at 300 DPI "for quality" (A4 → 2480×3508) is 4–9× wasted work through render/encode/transfer/decode — the model downscales to ~1024 px anyway. **Render to model-native size: long edge 1024–1600 px (~135 DPI).** This alone cuts upstream CPU and bytes ~75–85%.

**Tools:** pypdfium2 (Apache — the permissive workhorse) or pdftoppm/Poppler CLI (GPL-safe across process boundary); PyMuPDF/mutool are fastest but **AGPL** — the DeepSeek toolchain already drags PyMuPDF in (its own issue #223); get legal sign-off or avoid. `qpdf --split-pages` (<1 s for 200 pages) enables clean per-page CLI parallelism. **Process pools, never threads** (GIL + fitz thread-unsafety). ~0.6–1.5 s for 200 pages on 16 cores at target resolution.

**Format:** JPEG q90 via libjpeg-turbo (~120–300 KB/page; decode several× faster than PNG — this matters because *vLLM's frontend decodes every image on CPU*); grayscale for scan-only corpora; PNG only for lossless-compliance; never q≤75 for fine print; gzip on JPEG bodies wastes CPU. Heavy resizes: pyvips/OpenCV, not vanilla PIL.

**Base64, honestly accounted:** encode CPU is a non-issue (~1–2 GB/s C impl; ~40 ms for 200 pages). Real costs: +33% wire (irrelevant in-cluster; real over WAN to RunPod — prefer blob-URL fetch there), `json.dumps` copying megabyte strings (→ orjson + single `b"".join`), vLLM frontend buffering/parse/decode (→ `--api-server-count`, 8–16 vCPU), memory churn at 250 in-flight. Escape hatch in-cluster: `file://` + `--allowed-local-media-path` as a *sidecar* pattern (shared emptyDir on node NVMe) — never cross-node RWX file passing.

**Ranked bottleneck audit:** 1 GPU inference · 2 wrong raster resolution · 3 vLLM server-side CPU preprocessing (GPU sawtoothing <90% while gateway idles = frontend starving the engine) · 4 serial/threaded Python rasterization · 5 source-PDF download (200-page scans = 100–400 MB; xref at EOF ⇒ not streamable; ranged parallel GET + split-on-arrival) · 6 straggler pages · 7 event-loop blocking (CPU work inline in asyncio freezes all in-flight requests) · 8 HTTP plumbing (connection caps, per-request TLS, mesh mTLS on 67 MB of b64) · 9 per-page queue hops · 10 needless temp files.

**Gateway language:** Rust (tokio+rayon, pdfium-render, no GIL — the neural-maze reference uses Rust/Axum) > Go (80% of the benefit, 30% of the effort) ≈ .NET 8 (first-class Azure SDKs — a genuine argument on AKS) > Python-kept-honest (process pools + orjson + uvloop; fine for v1). Hybrid, not rewrite: systems gateway in Rust/Go/.NET, Python where the ML ecosystem lives; the pdftoppm-subprocess trick makes rendering language-neutral.

---

## 6. Deployment map & EU sovereignty

Four market shapes beyond hyperscalers: **neoclouds** (CoreWeave — Platinum ClusterMAX, priced near hyperscalers; Nebius/Lambda/Crusoe/Nscale ~10–15% cheaper, real managed K8s → manifests port), **serverless GPU** (RunPod, Modal — Python-native per-second, Koyeb — serverless containers scale-to-zero, Baseten, fal.ai), **marketplaces** (Vast.ai/Spheron — cheapest, never customer documents), **EU-sovereign** (the tier that matters).

**Sovereignty fine print:** CLOUD Act exposure follows *incorporation*, not server geography — a US cloud's EU region remains compellable. Non-US entities: Scaleway (French; strongest sovereign H100 availability ~€0.50–2.50/hr; Kapsule managed K8s; SecNumCloud in progress), OVHcloud (multi-region EU, SLAs — but SecNumCloud excludes GPU SKUs, and a 2026 Ontario ruling reached data-in-France via its Canadian entity: audit corporate structures), Hetzner (cheapest, German — no H100/A100 class), Gcore, Verda, STACKIT, IONOS, Exoscale. Nebius: EU DCs but structurally complex (ex-Yandex) — legal review, don't assume. 2026 note: Scaleway/OVH/Hetzner all raised prices in one quarter (Hetzner +30–37%) — re-quote.

**Posture:** (1) AKS = control plane + steady state (Service Bus, Blob, identity, layout tier); (2) Scaleway/OVHcloud = sovereign GPU burst for production documents; (3) RunPod/Modal/Vast = non-sensitive (bake-offs, synthetic, load tests). All portable: everything is vLLM-in-a-container behind an OpenAI API — multi-cloud is a LiteLLM route entry.

---

## 7. Hungarian strategy & fine-tuning

**Status board:** GLM-OCR model ❌ (8 languages, no hu; breaks on adjacent Latin languages) · PaddleOCR-VL ✅-probable (109 languages; verify appendix of arXiv 2510.14528) · dots.ocr ✅ (2nd on GlotOCR) · HunyuanOCR-1.5 ✅ (top open on MORE; most faithful) · DeepSeek-OCR-2 ⚠️ (weak multi-script, but Latin base + best FT tooling) · Qwen3-VL ⚠️ (32-language OCR, hu unverified) · **Qwen3.5-4B/9B ✅ front-runner (201 languages, native multimodal)** · LightOnOCR-2-1B ✅ (European-focused, cheap probe).

**The ő/ű test:** Hungarian's double-acute accents are unique and the canonical failure (silent ö/ü or õ/û substitution) — plain CER barely registers it while it corrupts every field. Probe protocol: 30–50 real pages, all candidates, three metrics — CER, word accuracy, **diacritic-specific error rate** (á/é/í/ó/ú/ö/ü/ő/ű confusions counted separately). Two days; may end the fine-tuning conversation.

**Fine-tuning (only if the probe demands):**
- Tooling: **Unsloth** (DeepSeek-OCR/2 official; Qwen3.5 day-1) ≫ LLaMA-Factory (GLM-OCR official tutorial) ≫ Paddle ecosystem. Unsloth's Persian demo: 200K-sample dataset, 88.26% absolute CER improvement, *most of the gain after 60 steps at batch 8 (~480 samples)* — on a harder (new-script) problem than Hungarian. Vendor demo; treat as mechanism-proof, not promise.
- Recipe template = Unlimited-OCR's own creation: **freeze the vision encoder, LoRA the decoder** (r=16–64, +projector, 1–3 epochs, fits one 24 GB card; merge adapter before serving).
- Dataset economics: ~2–5K pairs = meaningful, 10–30K = production, 50–100K = robustness. Sourcing cheats: (1) **born-digital PDFs = free labels** (text layer is ground truth; a few thousand documents ⇒ 10–30K pairs at zero annotation cost — build this pipeline regardless, it feeds the golden set too); (2) **synthetic rendering** (Wikipedia-hu/OSCAR-hu/own-domain text → varied fonts/layouts → augraphy degradation; 100K pages/afternoon; ~80/20 synthetic/real).
- **Anti-forgetting:** 10–30% original multilingual/table/formula replay in the mix, or the fine-tune destroys the skills that justified the base model.
- **Format-native fine-tuning** — the double win: train on targets in *the glmocr ResultFormatter's expected conventions* (text / strict HTML / bare LaTeX); one run buys the language AND the output contract; the adapter shim shrinks toward identity.

---

## 8. Layout stage (PP-DocLayout-V3): CPU vs GPU

Applies only to GLM-pipeline and PaddleOCR-VL (MinerU folds it in; DeepSeek/Unlimited/dots/Hunyuan are end-to-end). It's an RT-DETR-class detector — 10–50× cheaper than recognition; trivially cheap placed right, silent bottleneck when starved.

| Tier | Placement | ~Latency/pg | Economics | When |
|---|---|---|---|---|
| Minimal (default) | CPU sidecar with gateway; ONNX/OpenVINO INT8; **4–8 real vCPUs** + explicit thread counts | 150–400 ms | 8-vCPU ≈ $0.35–0.40/hr → 3–6 pages/s | 24 GB fleet; all batch |
| Optimal | Shared T4/L4 layout micro-service (batch endpoint, TensorRT) | 15–40 ms | One ~$0.50/hr T4 → 25–60 pages/s, feeds the whole H100 farm; **CC ≥ 8.0 applies to the VLM, not the detector — the banished T4s belong here** | Fleet demand > ~5 pages/s |
| High-end | `--layout-device cuda:0` on the VLM's GPU/slice, TensorRT | 5–15 ms | "Free" but spends ~0.5–1 GB of premium VRAM | Interactive lane; monolithic Paddle pipeline; RunPod |

CPU layout adds ~0.2–0.3 s/document vs GPU's ~0.01 s — meaningful for a 2-s interactive SLO, invisible in batch. Failure modes: default K8s CPU requests starving the GPU behind it; co-locating on a 1g.10gb slice and eating KV headroom. (Estimates are RT-DETR-class, not published V3 numbers; verify the ONNX/OpenVINO export path exists.)

---

## 9. The glmocr SDK as platform + the model swap

**Constraint (correct): keep the SDK, swap the model.** `OCRClient` calls a plain OpenAI endpoint; the SDK maps dotted config paths to env vars (`pipeline.ocr_api.api_host` → `GLMOCR_PIPELINE_OCR_API_API_HOST`) — repointing needs zero code, pure ConfigMap.

**Swap options:** **A** — name alias (`--served-model-name glm-ocr` on a Qwen serve): instant, degraded (prompt/format mismatch mangles `ResultFormatter`) — the smoke test & probe vehicle. **B (production)** — ~200-line adapter proxy: GLM dialect ↔ engineered per-region prompts + output normalization; SDK stays a pinned unmodified dependency; the anti-corruption layer where every future model slots in. **C** — subclass `OCRClient`: cleaner in-process, couples to young internals; only if B's hop measurably hurts (it won't at region granularity). **End-state:** B + format-native fine-tune (§7) → normalization shrinks toward identity.

**The neural-maze AKS reference decoded** (`k8s/aks/kustomization.yml`): 5 manifests (Rust worker, vLLM, Redis, KEDA, Service) + one hash-suffixed ConfigMap rolling both deployments; worker = unmodified glmocr SDK env-pointed at `ocr-vlm-service:8000`; vLLM serves `Qwen/Qwen3.5-4B` (`GPU_MEMORY=0.9, MAX_MODEL_LEN=16384, MAX_NUM_BATCHED_TOKENS=262144, MAX_NUM_SEQS=512`); workers micro-batch (4 jobs / 100 ms) and KEDA multiplies replicas — concurrency lives in replica count. It is Option A live on AKS. Copy-with-eyes-open: seqs=512/262K tokens is big-card tuning (→ ~32–64 on 24 GB); no adapter layer (format-drift unhandled); `managed-by: gemini-cli-agent` — agent-scaffolded course artifact: validate, don't venerate.

**CI guardrails for the three-way version surface (SDK/adapter/weights):** three region fixtures run against the live pair — table→parseable HTML, formula→bare LaTeX, text containing ő/ű.

---

## 10. The Qwen map & the recognition ladder

| Gen | Sizes | Notes |
|---|---|---|
| Qwen3-VL (2025) | 2B/4B/8B/32B + MoE | 32-language OCR; superseded for this slot |
| **Qwen3.5 (Feb–Mar 2026)** | 397B-A17B … 35B-A3B, 27B, **9B/4B/2B/0.8B** | **Native multimodal across the family — the "-VL" suffix is gone**; early fusion beats Qwen3-VL on visual understanding; **201 languages**; 262K ctx; MTP-trained; day-1 Unsloth/vLLM. The generation with the right sizes AND languages |
| Qwen3.6 (Apr 2026) | 27B + 35B-A3B only | Multimodal, agentic-coding focused; MTP 1.4–2.2×; **no small sizes**, zero OCR evidence |
| Qwen3.7 (May 2026) | Max/Plus | Proprietary — irrelevant |
| Qwen3.8 | Unconfirmed lineup | Coding claims on unreproducible harnesses; independent verdict: "no measured reason to re-download" |

Correction on record: from Qwen3.5 onward, no "-VL" suffix ≠ text-only — the naming inverted; `Qwen/Qwen3.5-4B` sees images.

**Three-arm ladder for the SDK slot:** 1) **Qwen3.5-4B default** (~9 GB BF16; 24 GB comfortable; 2g.20gb MIG; best out-of-box Hungarian odds); 2) **Qwen3.5-9B quality arm** (~18 GB BF16, FP8 halves it; 3g.40gb; escalation = one MIG-profile change); 3) **Qwen3.6-35B-A3B** only with H100 budget after both fail (3B-active decode speed at 35B quality, MTP back — but ~70 GB BF16 and coding-tuned). Price consciously: 4B is ~3–5× slower per region than the 0.9B specialists.

**Full bake-off roster:** PaddleOCR-VL-1.6, dots.ocr, HunyuanOCR-1.5, DeepSeek-OCR-2, LightOnOCR-2-1B, Qwen3.5-4B-behind-glmocr — on CER, word acc, diacritic ER, TableTEDS, latency.

---

## 11. Text-only LLMs downstream — slots and the hard ceiling

A text-only model cannot fill the recognition slot (no vision encoder; the SDK sends pixels). Legitimate slots: (1) **KIE over OCR'd markdown** — matches the markdown-canonical + separate-extraction contract; broad-language LLMs handle Hungarian field semantics well; (2) **post-OCR diacritic restoration** — constrained to diacritic-only edits + edit-distance guard, never on numeric/ID fields (unrestrained, it fluently "fixes" names and amounts).

The measured ceiling: two-stage OCR+LLM systems score **~0 on chart benchmarks (CharXiv 0.0 vs 81.8 end-to-end)** and collapse on visual QA — anything visual (charts, checkboxes, stamps, spatial layout) dies at the OCR→text boundary. Route page-level visual questions to a VLM or don't answer them.

---

## 12. Confidence engineering — a pipeline property

**None of the candidate VLMs natively emit calibrated confidence** — a real regression vs classic OCR (Tesseract/PP-OCRv6/Azure DI give per-word scores; Mistral OCR 4 exposes them via API). Engineer it as a pipeline property (model-swap-proof, like everything else):

1. **Layout detection scores** — free, already in the pipeline; first flag.
2. **vLLM token logprobs** per region/field (min/mean/P10) — open-weight advantage; token-probability confidence calibrates markedly better than verbalized self-reports (ECE 27.3 vs 42.7); ConfBench (arXiv 2608.01792) benchmarks exactly this split for document KIE.
3. **OCR-grounding** — highest value (EXTRACTCONF, arXiv 2606.24420): binary *value-in-OCR match* catches hallucinated fields; their full system fuses ~40 features via CatBoost into a calibrated score routing to automation vs human review at threshold τ — you need ~3 features (value-in-OCR + logprob-min + a validator) for most of it.
4. **Deterministic validators** — Hungarian specifics: adóazonosító/adószám checksums, IBAN check digits, date/VAT sanity. Perfectly calibrated for the fields that matter most.
5. **Cross-model agreement** — the PP-OCRv6 hallucination cross-check doubles as confidence.
6. **Calibrate on the golden set** (temperature scaling / isotonic; track ECE) → τ = the human-review routing knob.

Production corroboration: Extend (commercial IDP) is phasing *out* raw logprobs-confidence in favor of grounding + review agents — logprobs alone don't survive production. Watch item: MinerU-Diffusion emits per-position confidence natively (diffusion decoding) but needs a custom non-vLLM engine — re-check quarterly.

---

## 13. Open decisions & build order

**Still open (answer before building more):** workload volumes & page distribution · latency class (interactive/batch/both → two lanes) · residency confirmation (PII/GDPR → tier-2 provider choice) · day-1 single-model commitment · output contract sign-off · failure semantics (partial delivery + per-page manifest + idempotency keys + DLQ-with-page-image) · gateway language (by team composition).

**Build order:**
1. Golden set + harness (CER / word acc / diacritic ER / TableTEDS) — Rule 0.
2. Born-digital self-labeling pipeline (feeds eval *and* training).
3. Probe bake-off (6 candidates, 30–50 pages, 2 days) → pick ONE model.
4. AKS deployment: glmocr SDK worker + vLLM (winner) + Redis + KEDA, neural-maze shape with corrected sizing; CPU layout tier; CI fixtures.
5. Adapter shim (Option B) if the winner isn't GLM-format-native.
6. Confidence stack v1: layout scores + logprobs + value-in-OCR + Hungarian validators; calibrate; wire τ → review queue.
7. Fine-tune only if step 3 demands: Unsloth LoRA, format-native targets, 10–30K pairs, replay mix; re-run harness per checkpoint.
8. Scale-out: H100 MIG packing per §2; per-model LB pools (GIE/LiteLLM); sovereign burst tier contract.

---

## Caveats (the honest column)
- OmniDocBench is saturated and vendor-reported; independent multi-script/blind evals reorder everything — the golden set is the only leaderboard that counts.
- Throughput/latency figures (pages/s, the 1–2-min 200-page wave, layout tiers, in-flight multipliers, token-per-page budgets) are engineering estimates — validate on your document mix before any SLA.
- Hungarian membership in PaddleOCR-VL's 109 list and Qwen3-VL's 32-language OCR set: probable, unverified; Qwen3.5's "201 languages" is a vendor claim needing OCR-specific probing.
- Unsloth's Persian fine-tune numbers are a vendor demo on a different script; dataset-size tiers are heuristics.
- Licensing wrinkles: PyMuPDF/mutool AGPL (DeepSeek toolchain pulls it in); GLM-OCR = MIT weights + Apache layout; MinerU = custom Apache-based; verify each HF repo before shipping.
- EU pricing/certifications (SecNumCloud scope, Nebius structure, Q2-2026 hikes) are snapshots; audit corporate structures, not marketing.
- vLLM flag names and Qwen architectures (Gated DeltaNet hybrid) move between versions — pin images; the glmocr SDK is young — pin it, and let the adapter shim insulate you.
- RunPod API v2 is public beta — regenerate clients from `/v2/openapi.json`; v1 remains fallback.
- The neural-maze repo is an agent-scaffolded teaching artifact: sound patterns, not your numbers.
