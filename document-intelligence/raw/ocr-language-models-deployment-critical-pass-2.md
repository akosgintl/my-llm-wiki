# Language, Models, Confidence & the Deployment Map — a Critical Pass (Part 2)
*Companion to "Rasterization, Encoding & the Real Bottlenecks". Covers everything since: the open design decisions, GPU-cloud alternatives and EU sovereignty, the Hungarian-language pivot, fine-tuning strategy, the layout-stage economics, the glmocr-SDK model swap, the Qwen family map, downstream text-LLM roles, and confidence engineering. Numbers marked ~ are engineering ballparks; pricing and model claims are Feb–Aug 2026 snapshots — re-verify before committing.*

---

## 0. The critique that matters most, up front

**We designed the supply side before interrogating the demand side — and the demand side promptly invalidated a core choice.** Everything through the serving recipes was built for a hypothetical document mix. One fact ("Hungarian is 80% of the corpus") demoted GLM-OCR from default workhorse to niche tool in a single sentence, because its 8-language core doesn't include Hungarian. That's not a failure of the earlier work — it's the reason the earlier work insisted on swappable parts. The load-bearing conclusion of this whole phase:

**Rule 0: the golden set is the most valuable artifact in the project.** Every open question — which model, whether to fine-tune, which cloud, which confidence threshold, when a new Qwen release matters — resolves to "run it against the golden set and read the diacritic column." A 50–100 page labeled Hungarian set + an eval harness (CER, word accuracy, diacritic-specific error rate, TableTEDS) costs one engineer-week and converts every future decision from vibes to measurement. Nothing else in this document is worth doing before it.

**Rule 0.5: keep the pipeline the fixed point, make the model the replaceable part.** The glmocr SDK (layout + fan-out + assembly + formatting) is where durable engineering value lives; the recognition model behind its HTTP contract is a config value. This inversion — pipeline as platform, model as plugin — is what makes the version treadmill (Qwen ships a generation every ~2 months) irrelevant instead of exhausting.

---

## 1. The open design tree (what's decided, what's not)

Settled: model landscape, vLLM-only serving, 24 GB Ampere+ common minimum, H100 MIG packing, per-model LB pools, fan-out/hedging, raster/encoding pipeline, gateway-language shortlist, **glmocr SDK as the mandated pipeline**, and (by the Hungarian fact) the elimination of GLM-OCR-the-model as default.

Still open — the frontier from the grilling round, answer these before building more:
1. **Workload reality** — volume, page-length distribution, born-digital vs scanned ratio (partially answered: 80% Hungarian).
2. **Latency class** — interactive vs batch vs both; decides warm pools, hedging, and whether the one-wave H100 math matters at all.
3. **Data residency** — PII + EU processing obligations; decides the entire burst-capacity strategy (see §2).
4. **Day-1 scope** — one model + escape hatch, not the six-model menu. Every extra model is an eval harness, a failure catalog, an image to patch.
5. **Ground truth** — Rule 0. Non-negotiable, first.
6. **Output contract** — markdown canonical + separate extraction stage (recommended), vs in-model KIE.
7. **Failure semantics** — partial delivery + per-page status manifest + idempotency keys + DLQ-with-page-image. Nobody specs this; everybody ships it wrong.
8. **Gateway language** — resolves on team composition, not benchmarks.

---

## 2. The deployment map beyond Azure/AWS/GCP/RunPod

The 2026 GPU market has four shapes; the axis that matters for this project is **jurisdiction**, not price.

| Shape | Who | What you get | Fit |
|---|---|---|---|
| Hyperscale neoclouds | CoreWeave (Platinum-rated ClusterMAX, priced near hyperscalers), Nebius, Lambda, Crusoe, Nscale, Fluidstack | Real managed K8s/Slurm GPU clusters ~10–15% under CoreWeave; your vLLM manifests + MIG + KEDA port nearly unchanged | Sustained capacity at a fraction of Azure GPU pricing |
| Serverless GPU | RunPod, **Modal** (Python-native, per-second billing), **Koyeb** (serverless *containers*, scale-to-zero — cleanest map onto existing vLLM images), Baseten, fal.ai, Replicate | Function/container-call inference, pay-per-use | Bursty waves (the 200-page scenario); trial Modal + Koyeb against RunPod |
| Marketplaces | Vast.ai, Spheron | Cheapest rates anywhere, host reliability varies | Benchmarking only — never customer documents |
| **EU-sovereign** | Scaleway, OVHcloud, Hetzner, Gcore, Verda (ex-DataCrunch), STACKIT, IONOS, Exoscale | Non-US legal entities → outside CLOUD Act compulsion | **The production-document tier** |

**The sovereignty fine print people get wrong:**
- "EU region of a US cloud" ≠ sovereign. CLOUD Act exposure follows *incorporation*, not server location. Azure EU regions remain US-compellable.
- Scaleway (French, Iliad): strongest H100 availability in the sovereign class, ~€0.50–2.50/hr, managed Kubernetes (Kapsule) with GPU pools — the closest like-for-like AKS substitute; SecNumCloud in progress.
- OVHcloud: multi-region EU, enterprise SLAs — but SecNumCloud doesn't cover the GPU SKUs, and a 2026 Ontario ruling compelled its *Canadian entity* to hand over data stored in France. Verify the full corporate structure, always.
- Hetzner: cheapest, German — but no A100/H100; workstation-RTX class only. Fine for a 0.9–4B replica, not the H100 tier.
- Nebius: EU datacenters, competitive H100 pricing — but structurally complex (ex-Yandex, multi-jurisdiction). Don't assume it matches OVHcloud/Hetzner without legal review.
- 2026 reality check: OVHcloud, Scaleway *and* Hetzner all raised prices in one quarter (Hetzner +30–37%, citing DRAM +171% YoY). The EU-discount era is ending; re-quote before budgeting.

**Recommended posture — three tiers:**
1. **AKS = control plane + steady state.** Service Bus, Blob, identity, the layout tier, and baseline GPU capacity stay put.
2. **Scaleway/OVHcloud = sovereign GPU burst** for production documents (EU entity, real K8s, manifests port).
3. **RunPod/Modal/Vast = non-sensitive tier** — bake-offs, synthetic data, load tests.

The connective tissue: everything is plain vLLM-in-a-container behind an OpenAI API. Multi-cloud here is a LiteLLM route entry, not a re-architecture. That was the point of building it this way.

---

## 3. The Hungarian pivot — model status and the test that matters

80% Hungarian re-ranks every model. Status board:

| Model | Hungarian status | Verdict |
|---|---|---|
| GLM-OCR (model) | 8 core languages, none Hungarian; independent tests show breakdowns on adjacent Latin languages (Polish) | ❌ Disqualified as default (SDK stays — see §6) |
| PaddleOCR-VL-1.6 | 109 languages (list in tech-report appendix, arXiv 2510.14528); classic PaddleOCR has long shipped `hu` | ✅ Likely — verify appendix, then test anyway |
| dots.ocr | 100+ languages, 2nd on GlotOCR mid-resource benchmark | ✅ Serious candidate |
| HunyuanOCR-1.5 | Top open model on MORE multilingual benchmark; most "faithful" (least prone to correcting the source) | ✅ Candidate |
| DeepSeek-OCR/2 | ~100 languages claimed; near-bottom on independent multi-script evals — but Hungarian is Latin-script, so the floor is higher | ⚠️ Probe it; strong fine-tune base (§4) |
| Qwen3-VL-4B/8B | OCR in 32 languages (39 tested, >70% acc on 32) — Hungarian membership unverified | ⚠️ Probe it |
| **Qwen3.5-4B / 9B** | **201 languages/dialects** (up from 119), native multimodal | ✅ **Front-runner** (§7) |
| LightOnOCR-2-1B | Explicitly European-language focused | ✅ Cheap probe, worth one afternoon |

**The ő/ű test.** Hungarian's double-acute accents are unique to the language and are the canonical OCR failure: silent substitution of ö/ü or õ/û. Plain CER barely registers it (single-character errors) while it corrupts every downstream field and search index. Therefore the probe protocol measures three numbers, and the third is the tiebreaker:
1. CER (overall)
2. Word accuracy
3. **Diacritic-specific error rate** — count á/é/í/ó/ú/ö/ü/ő/ű confusions as their own metric.

Probe protocol: 30–50 pages from the *real* corpus (mixed born-digital/scanned), run all candidates via their cheapest inference path, two days of work. It may end the fine-tuning conversation before it starts — if the best off-the-shelf CER is already low-single-digit with clean diacritics, ship it and move on.

---

## 4. Fine-tuning: tooling, dataset economics, and the two cheats

**Tooling maturity ranking (this decides more than architecture does):**

| Path | Maturity | Notes |
|---|---|---|
| **DeepSeek-OCR/2 via Unsloth** | ★★★ | Official support, free Colab notebooks (+eval variant), 1.4× faster / −40% VRAM. The existence proof: their Persian demo — 200K-sample dataset, **88.26% absolute CER improvement, with most of the gain after 60 steps at batch 8 (~480 samples seen)**. Persian is a *new script*; Hungarian (Latin + new diacritics) is strictly easier |
| **Qwen3.5 via Unsloth / LLaMA-Factory** | ★★★ | Day-1 framework support across the family; first-class transformers integration, no custom-code surgery |
| GLM-OCR via LLaMA-Factory | ★★ | Official tutorial (Feb 2026) — but you'd be teaching a language it barely has, from the worst starting point |
| PaddleOCR-VL | ★ | Real but Paddle-ecosystem-native; friction if the team isn't |
| Unlimited-OCR | — | No tooling, but it *is* the recipe template: Baidu made it by continue-training DeepSeek-OCR for 4,000 steps, **vision encoder frozen, decoder only**. Hungarian is a language problem, not a vision problem — freeze the encoder, LoRA the decoder |

**Dataset sizing (Latin-script adaptation of a 3–4B decoder):** meaningful gains ~2–5K page/region pairs; solid production adaptation 10–30K; robustness across fonts/degradation 50–100K+. But sourcing beats sizing, via two cheats:

1. **Born-digital PDFs are free labels.** Text layer = ground truth: render page → image, pair with extracted text. A few thousand born-digital Hungarian documents ⇒ the 10–30K training set costs zero annotation hours. Build this pipeline regardless of the fine-tune decision — it also feeds the golden set.
2. **Synthetic rendering scales the tail.** Hungarian corpora (Wikipedia-hu, OSCAR-hu, Hungarian National Corpus, own-domain text) → rendered pages with varied fonts/layouts → degraded with augraphy (noise, blur, skew) to imitate the scanned fraction. 100K pages ≈ one afternoon of compute. Mix ~80% synthetic / 20% real.

**The anti-forgetting clause:** keep 10–30% of the original multilingual/table/formula distribution in the training mix, or the fine-tune will destroy the table/layout skills that justified the base model.

**The move that dissolves two problems at once — format-native fine-tuning.** Since a Hungarian LoRA is likely anyway, make the training targets *the glmocr ResultFormatter's expected output conventions* (region crop → text / strict HTML tables / bare LaTeX). One training run buys the language AND the output contract; the adapter shim (§6) shrinks to nearly nothing; conformity is guaranteed by weights instead of prompt engineering.

Mechanics: LoRA r=16–64 on the decoder (+projector), encoder frozen, 1–3 epochs, fits one 24 GB card (the existing common-minimum GPU) under Unsloth; merge the adapter before serving so the vLLM recipes stay untouched; gate every checkpoint on the held-out golden set *with the diacritic metric*, not CER alone.

Honesty note: the Unsloth Persian numbers are the vendor's own demo on a different script. Treat "60 steps fixed it" as proof the mechanism works, not a promise about this corpus.

---

## 5. Layout-stage economics (PP-DocLayout-V3): CPU vs GPU

Only GLM-OCR-pipeline and PaddleOCR-VL deployments have this decision (MinerU folds layout into its own VLM; DeepSeek/Unlimited/dots/Hunyuan are end-to-end). The asymmetry that frames everything: PP-DocLayout-V3 is an RT-DETR-class *detector* — 10–50× cheaper per page than recognition. It's trivially cheap when placed right and the silent bottleneck when starved.

| Tier | Placement | ~Latency/page | ~Cost profile | When |
|---|---|---|---|---|
| **Minimal** | CPU, co-located with gateway; ONNX Runtime / OpenVINO INT8, 4–8 real vCPUs | ~150–400 ms | 8-vCPU pod ≈ $0.35–0.40/hr → 3–6 pages/s | 24 GB fleet; all batch lanes. **The default** |
| **Optimal** | One shared small GPU (T4 ~$0.50/hr) as a layout micro-service | ~15–40 ms | ~25–60 pages/s — one card feeds the entire 7-replica H100 farm | Fleet demand > ~5 pages/s sustained. Note: the CC ≥ 8.0 floor applies to the *VLM under vLLM*, not to a detector — the banished T4s are exactly right here |
| **High-end** | Same GPU/MIG slice as the VLM (`--layout-device cuda:0`), TensorRT-compiled | ~5–15 ms | "Free" but spends VLM VRAM (~0.5–1 GB) and SM time on your most expensive silicon | Interactive lane; monolithic PaddleOCR pipeline; RunPod (can't rent fractional T4s) |

Failure modes, ranked: (1) default K8s CPU requests — 2 vCPU layout pods starving an H100; give layout real cores and explicit thread counts. (2) Rendering the layout decision at 300 DPI (same resolution critique as Part 1). (3) Co-locating on a 1g.10gb slice where the VRAM bite forces `gpu-memory-utilization` down and eats KV headroom.

Latency accounting: CPU layout adds ~0.2–0.3 s per document vs GPU's ~0.01 s — meaningful against a 2-second interactive SLO, invisible in batch. Caveat: these are RT-DETR-class engineering estimates, not published PP-DocLayout-V3 benchmarks; and verify the ONNX/OpenVINO export path exists for V3 specifically before betting Tier 1 on it.

---

## 6. The glmocr SDK as fixed point + the model swap

**The constraint (correct one): keep the SDK, change the model.** The SDK is where non-model value lives — PP-DocLayout-V3 integration, parallel region fan-out, reading-order assembly, JSON/Markdown formatting, Flask server, `layout_device` placement. Its `OCRClient` calls a plain OpenAI-compatible endpoint (`pipeline.ocr_api.api_host/port`), which makes the model a config value.

**The killer discovery (from the neural-maze repo): env-var config override.** The SDK maps dotted config paths to `GLMOCR_`-prefixed env vars — `pipeline.ocr_api.api_host` → `GLMOCR_PIPELINE_OCR_API_API_HOST`. Repointing the SDK at any vLLM service requires zero code, zero YAML edits — pure Kubernetes ConfigMap. This is the cleanest possible integration surface.

**Three swap options, ranked:**
- **A — name alias, zero code** (`vllm serve Qwen/Qwen3.5-4B --served-model-name glm-ocr`): runs end-to-end immediately. Quality degrades because GLM-OCR's fixed task prompts meet a different model's output dialect (preambles, code fences, QwenVL-HTML) and `ResultFormatter` mangles it. **The smoke test and probe vehicle, not production.**
- **B — adapter proxy (recommended production shape):** ~200-line shim speaking "GLM dialect" to the SDK and engineered per-region-type prompts to the model, normalizing responses back. SDK stays a pinned, unmodified pip dependency; the adapter is also where future models (Hunyuan, fine-tuned checkpoints) slot in. The anti-corruption layer.
- **C — subclass `OCRClient` in-process:** the package is explicitly composable, but couples you to young internal interfaces. Only if B's network hop measurably hurts (at region granularity it won't).
- **End-state: A½/B + format-native fine-tune (§4)** — the adapter's normalization shrinks toward identity.

**The neural-maze AKS reference, decoded** (`k8s/aks/kustomization.yml`): five manifests (Rust API/worker, vLLM, Redis, KEDA scaler, Service) + one shared `configMapGenerator` (hash-suffixed → config changes roll both deployments). Worker = unmodified glmocr SDK pointed via env vars at `ocr-vlm-service:8000`; vLLM serves `SERVED_NAME=Qwen/Qwen3.5-4B` with `GPU_MEMORY=0.9, MAX_MODEL_LEN=16384, MAX_NUM_BATCHED_TOKENS=262144, MAX_NUM_SEQS=512`; workers micro-batch (`MAX_BATCH_SIZE=4`, `BATCH_WINDOW_MS=100`) and KEDA multiplies replicas on queue depth — concurrency lives in replica count, not per-worker parallelism. It is exactly Option A, live on AKS.

Copy-with-eyes-open list: `MAX_NUM_SEQS=512` + 262K batched tokens is big-card tuning — scale to ~32–64 on the 24 GB fleet or it OOMs. No adapter layer exists — the format-drift risk is unhandled. `managed-by: gemini-cli-agent` — it was scaffolded by an agent; validate, don't venerate. Three region-level CI fixtures guard the whole stack: a table region (parseable HTML), a formula region (bare LaTeX), a text region containing ő/ű — because SDK, adapter, and weights now version independently and any of them can silently break the contract.

---

## 7. The Qwen map and the recognition-candidate ladder

**Family status (Aug 2026):**

| Generation | Sizes | Multimodal | Character | For this project |
|---|---|---|---|---|
| Qwen3-VL (2025) | 2B/4B/8B/32B dense + MoE | Yes (suffix era) | OCR in 32 languages | Superseded by 3.5 for this slot |
| **Qwen3.5 (Feb–Mar 2026)** | 397B-A17B → 122B-A10B, 35B-A3B, 27B, **9B, 4B, 2B, 0.8B** | **Native, whole family — no "-VL" suffix exists anymore**; early fusion beats Qwen3-VL on visual understanding | **201 languages**, 262K context, MTP-trained, day-1 Unsloth/vLLM | **The generation with the right sizes and the right languages** |
| Qwen3.6 (Apr 2026) | 27B dense + 35B-A3B only | Yes | Agentic-coding focused; MTP serving confirmed (NEXTN) | No small sizes; coding gains, zero OCR evidence |
| Qwen3.7 (May 2026) | Max/Plus | — | Proprietary API | Irrelevant |
| Qwen3.8 (recent) | Unconfirmed; no small tier observed | — | Coding claims on unreproducible harnesses; "no measured reason to re-download if you run 3.6" (independent review) | Watch, don't adopt |

Correction on the record: "no -VL suffix" ≠ text-only from Qwen3.5 onward — the naming convention inverted, and the earlier "a non-VL model can't fill the slot" answer, while right in principle, was wrong about `Qwen/Qwen3.5-4B` specifically. The version treadmill is real (~2-month cadence); Rule 0.5 is the cure — any new checkpoint is one `SERVED_NAME` change + one golden-set run from a verdict.

**The three-arm ladder for the SDK's recognition slot:**
1. **Qwen3.5-4B (default):** ~9 GB BF16 → comfortable on 24 GB, 2g.20gb MIG. 201-language coverage = best out-of-box Hungarian odds of any candidate. What the reference repo runs.
2. **Qwen3.5-9B (quality-control arm):** ~18 GB BF16 (tight on 24 GB; FP8 halves it), 3g.40gb MIG. Escalation costs one MIG-profile change.
3. **Qwen3.6-35B-A3B (escalation, H100-only, only if both fail):** 3B-active MoE = small-model decode speed at 35B quality, MTP 1.4–2.2× — but ~70 GB BF16 and coding-tuned training makes OCR behavior an open question.

Full bake-off roster: PaddleOCR-VL-1.6, dots.ocr, HunyuanOCR-1.5, DeepSeek-OCR-2, LightOnOCR-2-1B, Qwen3.5-4B-behind-glmocr. Measured on: CER, word accuracy, diacritic ER, TableTEDS, per-page latency. Trade-off to price consciously: 4B is ~3–5× slower per region than the 0.9B specialists — language coverage isn't free.

---

## 8. Text-only LLMs: the right slots and the hard ceiling

A text-only model (Qwen3-8B-class) cannot occupy the recognition slot — no vision encoder, and the SDK sends pixels. Its legitimate slots are downstream:

1. **KIE over OCR'd markdown** — matches the recommended output contract (markdown canonical + separate extraction). Broad-language text LLMs handle Hungarian field semantics well.
2. **Post-OCR diacritic restoration** — tempting (*kör* vs *kőr* is a language-model problem), **dangerous unrestrained**: it will fluently "fix" names, amounts, IDs. If used: diacritic-only edit constraint + edit-distance guard + never on numeric/ID fields.

The ceiling, measured: in Qianfan's comparison, two-stage OCR+Qwen3-4B systems scored **~0 on chart benchmarks (CharXiv 0.0 vs 81.8 end-to-end)** and collapsed on visual QA generally. Anything *visual* — charts, checkboxes, stamps, spatial layout — dies at the OCR→text boundary. Fields that survive as text: fine. Questions about the page: route to a VLM or don't answer.

---

## 9. Confidence engineering: a pipeline property, not a model feature

**The uncomfortable baseline: none of the candidate VLMs natively emit calibrated confidence.** This is a genuine regression vs classic OCR (Tesseract, PP-OCRv6, Azure Document Intelligence: per-word confidence; Mistral OCR 4 exposes it via API). Autoregressive decoders emit tokens, not certainty. Therefore confidence must be *engineered as a pipeline property* — which, conveniently, makes it model-swap-proof like everything else.

**The signal stack, cheapest first:**
1. **Layout detection scores** — PP-DocLayout-V3 boxes carry confidence already. Free. First flag.
2. **vLLM token logprobs per region/field** — request `logprobs`, aggregate min/mean/P10 per field. Open-weight-on-vLLM means you get the *better* estimation family for free: token-probability confidence calibrates markedly better than verbalized self-reports (ECE 27.3 vs 42.7 in the calibration literature); ConfBench (arXiv 2608.01792) benchmarks exactly this verbalized-vs-logprob split for document KIE.
3. **OCR-grounding checks** — the highest-value signal per EXTRACTCONF (arXiv 2606.24420): a binary *value-in-OCR match* (does the extracted value appear anywhere in the OCR text?) catches hallucinated fields with no grounding; their full system fuses ~40 features (logprob stats, entropy, cross-call agreement, spatial divergence) through CatBoost into a calibrated score routing each field to automation or human review at threshold τ. You need ~3 of the 40 to get most of it: value-in-OCR + logprob-min + a validator.
4. **Deterministic validators** — for Hungarian business documents specifically: adóazonosító/adószám checksums, IBAN check digits, date-format and VAT-rate sanity. *Perfectly* calibrated for the fields that matter most, regex-cheap.
5. **Cross-model agreement** — the PP-OCRv6 hallucination cross-check doubles as a confidence signal.
6. **Calibration on the golden set** (temperature scaling / isotonic; track ECE) → τ becomes the human-review routing knob, closing the HITL question.

**Production reality check:** Extend (commercial IDP) has been *phasing out* raw logprobs-confidence on newer processors in favor of OCR-grounding + a review agent — corroborating the research: logprobs alone don't survive production; grounding + validators + calibration do.

**Watch item:** MinerU-Diffusion decodes by iteratively unmasking positions on per-position confidence — a diffusion OCR decoder emits confidence *natively* as a byproduct. Incompatible with standard vLLM (custom engine); a curiosity to re-check quarterly, not a plan.

---

## TL;DR decision card
1. **Golden set first** (50–100 real pages, CER + word acc + diacritic ER + TableTEDS harness). Every subsequent decision is a query against it.
2. **Deployment = three tiers:** AKS control plane · Scaleway/OVHcloud sovereign GPU burst (CLOUD Act follows incorporation, not geography) · RunPod/Modal for non-sensitive work. All portable because everything is vLLM behind an OpenAI API.
3. **Hungarian:** GLM-OCR-the-model out; **Qwen3.5-4B behind the glmocr SDK is the front-runner** (201 languages, right size, day-1 fine-tune tooling); PaddleOCR-VL-1.6 and dots.ocr as the specialist arms; the ő/ű diacritic metric is the tiebreaker.
4. **Fine-tune only if the probe demands it:** Unsloth LoRA, encoder frozen, born-digital self-labeled pairs + synthetic rendering, 10–30K pairs, 10–30% replay data, **format-native targets** so one run buys language + output contract.
5. **Layout stage:** CPU/OpenVINO default → shared T4 micro-service past ~5 pages/s → same-GPU only where topology demands.
6. **SDK swap:** env-var repointing (zero code) for probes; adapter shim for production; three CI fixtures (table-HTML, formula-LaTeX, ő/ű text) guard the three-way SDK/adapter/weights version surface.
7. **Confidence is engineered, not emitted:** layout scores + logprobs + value-in-OCR grounding + Hungarian checksums, calibrated on the golden set, τ routes to human review.
8. **Ignore the Qwen version treadmill** — 3.6/3.8 are coding releases with no small sizes; a new checkpoint earns adoption via the golden set, never via the release blog.

## Caveats
- All EU pricing/certification claims (Scaleway SecNumCloud, Nebius structure, the Q2-2026 price hikes) are snapshots — re-verify the two shortlisted providers before contracts.
- Hungarian membership in PaddleOCR-VL's 109-language list and Qwen3-VL's 32-language OCR set is *probable, unverified* — check the respective tech-report appendices; Qwen3.5's "201 languages" is a vendor claim covering the text side and needs OCR-specific probing like everything else.
- Unsloth's Persian fine-tuning numbers are a vendor demo on a different script; the dataset-size guidance (2–5K/10–30K/50–100K) is engineering heuristic, not literature-backed for Hungarian specifically.
- Layout-stage latencies are RT-DETR-class estimates, not published PP-DocLayout-V3 benchmarks; the ONNX/OpenVINO export path for V3 needs verification.
- The neural-maze repo is an agent-scaffolded course artifact, not a battle-tested reference — its patterns (env-var config, KEDA shape) are sound; its numbers are not yours.
- Qwen3.8's size lineup was unconfirmed at time of writing; the "no measured reason" verdict is one independent reviewer's.
- The glmocr SDK is young: pin its version, and expect the env-var convention and internal interfaces to churn — the adapter shim (Option B) is your insulation.
