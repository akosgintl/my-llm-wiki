# Open-Source OCR / Visual Document Understanding Models for Self-Hosted AKS Pipelines: A Comparative Investigation (August 2026)

## TL;DR
- **For an event-driven, SLM-powered OCR/VDU pipeline on AKS with NVIDIA GPUs, no single model wins everything — route by workload.** GLM-OCR (0.9B, MIT, ~3 GB VRAM, 1.86 pages/s) is the best default for single-page invoices/receipts/KIE and high-throughput batch on commodity GPUs; PaddleOCR-VL-1.6 (0.9B, Apache-2.0) is the most robust for multilingual/messy/in-the-wild documents; Baidu Unlimited-OCR (3B MoE, MIT) is uniquely suited to long multi-page contracts via one-shot parsing; MinerU (custom Apache-based license) is the best full-stack ingestion platform.
- **Treat vendor OmniDocBench scores skeptically.** GLM-OCR (94.62), PaddleOCR-VL-1.6 (96.33) and MinerU2.5-Pro (95.69) crowd the top of the vendor leaderboard, but the benchmark is saturated and Latin/CJK-print-biased; independent multi-script and blind-voting evaluations reorder the field substantially and DeepSeek-OCR/MinerU rank near the bottom on hard real-world scripts.
- **Deployment practicality varies more than accuracy.** GLM-OCR, PaddleOCR-VL and dots.ocr are cleanly vLLM/SGLang-served and fit on L4/A10 GPUs; DeepSeek-OCR and Unlimited-OCR require custom n-gram logits processors and disabled prefix caching, and Unlimited-OCR additionally needs FlashAttention-3 (Hopper) for its long-context path and a bundled dev-build SGLang wheel — meaningful AKS friction.

## Key Findings

**The landscape as of August 2026.** Document OCR has converged on compact (0.9B–3B) specialized vision-language models that emit structured Markdown/JSON directly, largely replacing multi-stage detect-then-recognize pipelines. The most-cited benchmark, OmniDocBench (now v1.6, CVPR 2025 origin, OpenDataLab/Shanghai AI Lab), is widely acknowledged as saturated — the top open models cluster within ~2 points at 94–96%.

**Vendor OmniDocBench v1.5/v1.6 leaderboard (overall, vendor-reported):**
- PaddleOCR-VL-1.6: 96.33 (v1.6), released May 28, 2026
- MinerU2.5-Pro: 95.69 (v1.6 Full)
- GLM-OCR: 94.62 (v1.5) / 95.15–95.22 (v1.6)
- PaddleOCR-VL-1.5: 94.5 (v1.5) / 94.87–94.93 (v1.6)
- Baidu Unlimited-OCR: 93.23 (v1.5) / 93.92 (v1.6)
- DeepSeek-OCR 2: 91.09 (v1.5)
- MinerU2.5: 90.67 (v1.5) / 92.98 (v1.6)
- dots.ocr: ~88.4 (v1.5)
- DeepSeek-OCR (v1): 87.01 (v1.5)
- HunyuanOCR-1.5: 94.74 (v1.6)

**Independent evaluations tell a different story.** On Noah Dasanaike's (Harvard) independent "socOCRbench" (280 full-page multi-region/multi-script images; overall = mean of NES, chrF, TEDS; updated through June 2026), the ranking inverts for models overfit to clean print: dots.ocr 1.5 scored ~0.478, PaddleOCR-VL-1.6 ~0.394, GLM-OCR ~0.368, but DeepSeek-OCR2 only ~0.176, MinerU2.5-Pro ~0.165, and DeepSeek-OCR v1 ~0.086 (below Tesseract v5 at ~0.098); the best proprietary system, Gemini 3.1 Pro (low), topped the board at 0.6357. On the blind community ELO leaderboard OCR Arena (ocrarena.ai, built by Extend.ai), dots.ocr ranks #12 (ELO 1442) and "outperforms GLM-OCR with a 88.9% win rate across 9 matchups," while GLM-OCR holds #24 (ELO 1347) and olmOCR 2 ranks #19 (ELO 1382), beating GLM-OCR with a 92.3% win rate across 13 matchups. On the olmOCR-bench (allenai), the open specialists cluster ~75–80%: PaddleOCR-VL ~80.0, DeepSeek-OCR ~75.7, DeepSeek-OCR2 ~76.3, dots.ocr ~79.1, GLM-OCR ~75.2, with GLM-OCR notably weak on the Long/Tiny-text category (~35.7). On the mid-resource multi-script GlotOCR Bench (arXiv 2604.12978), "GLM-OCR and DeepSeek-OCR-2 perform over 40 points below Gemini 3.1 Flash-Lite," which leads (CER 27.7, Acc@5 61.9%) with dots.ocr second.

**Repetition loops are the dominant production failure mode across the DeepSeek-OCR lineage.** Jim Clifford (University of Saskatchewan) reported in DeepSeek-OCR GitHub Issue #151, after evaluating on 600 British Library historical newspaper clippings (1800s–1900s), "a persistent 9.2% catastrophic failure cohort" (loops/duplication) despite guardrails. DeepSeek's own OCR-2 paper (arXiv 2601.20552, §5.3) states it is "reducing the repetition rate from 6.25% to 4.17% for online user-log images, and from 3.69% to 2.88% for PDF data production" — measured because ground truth is unavailable in production. dots.ocr documents endless-repetition on continuous special characters (ellipses/underscores) and does not parse in-document pictures. This is why DeepSeek-OCR and Unlimited-OCR ship with mandatory n-gram logits processors.

## Details (per-model deep dives)

### 1. DeepSeek-OCR (Oct 2025) and DeepSeek-OCR 2 (Jan 27, 2026)
- **Architecture:** DeepEncoder (SAM-ViT-base + CLIP-large, ~380M, 16× convolutional token compression) + DeepSeek-3B-MoE decoder (~570M active). OCR-2 replaces the CLIP-style encoder with DeepEncoder V2 — a Qwen2-0.5B-based "Visual Causal Flow" encoder that reorders visual tokens into a learned human-like reading order; keeps the 3B-MoE (~500M active) decoder. 256–1120 visual tokens/page.
- **Benchmarks (vendor):** v1 = 87.01, OCR-2 = 91.09 on OmniDocBench v1.5 (+3.73). Reading-order edit distance improved 0.085→0.057. Fox benchmark: 97% accuracy at 10× optical compression, ~60% at 20×.
- **Throughput/hardware:** ~200,000 pages/day on a single A100-40G (vendor); ~0.1–0.4 s/page. VRAM ~6.7 GB BF16 (fits 16 GB comfortably; runs on RTX 4090/L4/A10); GGUF quants down to ~4 GB (Q4). MoE reduces per-token cost.
- **Serving:** vLLM (official recipe with `NGramPerReqLogitsProcessor`, `--no-enable-prefix-caching`, `--mm-processor-cache-gb 0`), Transformers (`trust_remote_code`, flash_attention_2), Ollama, llama.cpp (GGUF). OCR-2 vLLM support lagged behind release (required a specific PR). deepseek-ocr.rs offers a Rust multi-backend port.
- **Formats:** Markdown, grounding/bounding boxes (`<|grounding|>`), tables, LaTeX formulas, chart/chemical-formula parsing, 100+ languages.
- **License:** v1 code MIT; OCR-2 repo Apache-2.0 (note the repo pulls PyMuPDF/fitz which is AGPL — a flagged licensing wrinkle for redistribution). Model weights permissive.
- **Limitations:** Repetition/hallucination on long/dense/handwritten/non-English/vertical text; "linguistic crutch" behavior (corrects visually-clear but semantically-odd words); newspapers >0.13 edit distance. Near-bottom on independent multi-script benchmarks. Last place historically on OCR Arena for v1.
- **AKS verdict:** Strong for high-volume batch of clean digital PDFs where cost/page matters; budget for repetition retries.

### 2. GLM-OCR (Zhipu/Z.ai, Feb 2026; tech report arXiv 2603.10910, Mar 2026)
- **Architecture:** 0.9B total = 0.4B CogViT visual encoder + 0.5B GLM decoder. Multi-Token Prediction (MTP) head predicts ~10 tokens/step (avg 5.2 realized, ~50% throughput gain) and doubles as a speculative-decoding draft head at serving time. Two-stage pipeline: PP-DocLayout-V3 layout analysis → parallel region-level recognition. Trained with GRPO RL (rewards: NED, CDM, TEDS, field-F1, repetition/malformed-structure penalties).
- **Benchmarks (vendor):** #1 on OmniDocBench v1.5 at 94.62 (beats Gemini-3 Pro 90.33, Qwen3-VL-235B 89.15, GPT-5.2 85.4). Table TEDS 93.96, formula CDM 93.90/UniMERNet 96.5, OCRBench-Text 94.0. Strong KIE: Nanonets-KIE 93.7, Handwritten-KIE 86.1. In-house wins on seals (90.5 vs dots.ocr 63.0), receipts (94.5).
- **Throughput/hardware:** 1.86 PDF pages/s, 0.67 images/s (fastest in class, single replica/concurrency). ~2.9–3 GB VRAM at FP16 (~0.7 GB INT4); runs on RTX 4090/L4/A10 and even CPU; ~2.5 GB in practice.
- **Serving:** vLLM (with `--speculative-config '{"method":"mtp"}'`), SGLang (NEXTN speculative), Ollama, Apple MLX, hosted MaaS API. Requires transformers ≥5.3 and (at launch) vLLM nightly. Official SDK + agent "Skill" mode.
- **Formats:** Semantic Markdown, HTML tables, LaTeX, JSON KIE via schema. 8 primary languages (zh, en, fr, es, ru, de, ja, ko); ~100+ claimed internally.
- **License:** Model weights MIT; bundled PP-DocLayout-V3 Apache-2.0. Fully commercial.
- **Limitations:** Prompt rigidity (small fixed prompt set; no free-form Q&A); weak on long/tiny text (olmOCR-bench ~35.7); two-stage error propagation; stochastic whitespace/line-break variation; limited beyond its 8 core languages (hangs/gibberish reported on Hindi/Arabic/Polish/French in one hands-on test) and >40 points below Gemini 3.1 Flash-Lite on mid-resource scripts (GlotOCR Bench). Independent blind voting ranks it below dots.ocr and olmOCR-2.
- **AKS verdict:** Best default for single-page KIE, receipts/invoices, seals, and high-throughput batch on cheap GPUs. Cleanest small-model serving story.

### 3. Baidu Unlimited-OCR (June 22, 2026; arXiv 2606.23050)
- **Architecture:** Continue-trained from the DeepSeek-OCR checkpoint (~4,000 steps, decoder-only, frozen DeepEncoder, ~2M doc samples on 8×16 A800). Same DeepEncoder + 3B-MoE (500M active) decoder, but every attention layer replaced with **Reference Sliding Window Attention (R-SWA)**: each generated token attends to all visual/prompt "reference" tokens permanently, plus only the last ~128 output tokens. KV cache is bounded (O(1)) regardless of output length → flat memory and latency for long documents.
- **Benchmarks (vendor):** 93.23 (v1.5, +6.22 over DeepSeek-OCR baseline; beats DeepSeek-OCR2's 89.17 in the same table), 93.92 (v1.6). Long-horizon: edit distance <0.11 with 40+ pages input, 97% Distinct-35.
- **Throughput/hardware:** ~35% faster than DeepSeek-OCR at 6,000 generated tokens (TPS 7,847 vs 5,823), gap widening with length. 3B/500M-active; single 8–12 GB+ GPU for BF16; runs on RTX 4090. 32K max context; multi-page batching limited to "base" (1024) mode.
- **Serving:** Transformers (`trust_remote_code`), vLLM (dedicated image `vllm/vllm-openai:unlimited-ocr` + `NGramPerReqLogitsProcessor`, prefix caching/mm-cache disabled), SGLang (bundled dev-build wheel, `--attention-backend fa3` [FlashAttention-3, Hopper-only], `--page-size 1`, `--enable-custom-logit-processor`, `--disable-overlap-schedule`). llama.cpp GGUF via community `unlocr`. Two modes: gundam (640, crop, fast single-image) and base (1024, no-crop, multi-page). >1M HF downloads in first two weeks.
- **Formats:** Markdown + layout info; end-to-end (no per-word boxes/confidence). Inherits DeepSeek multilingual/chart/formula capabilities.
- **License:** MIT weights + code.
- **Limitations:** Not truly unlimited (32K bounds prefill); multi-page uses base mode only, so very small text can be missed; no bounding boxes; likely inherits DeepSeek repetition tendencies; little independent evaluation exists yet. The FA3/Hopper requirement and dev-build SGLang wheel are real AKS deployment gotchas.
- **AKS verdict:** The standout choice for long multi-page contracts/reports/books where cross-page structure and "no splitting/merging" matter — but plan for Hopper-class GPUs (H100/H200) to use the long-context serving path optimally. This exact model+AKS pattern is used in a public production-OCR reference course (neural-maze), which deploys Unlimited-OCR behind vLLM on AKS with a Rust/Axum gateway, Redis, and KEDA scale-to-zero.

### 4. PaddleOCR-VL / -1.5 / -1.6 (Baidu; 0.9B, ERNIE-4.5-0.3B based)
- **Architecture:** Two-stage: PP-DocLayoutV3 (RT-DETR layout, multi-point/polygonal localization for skew/warp, Global-Pointer reading order) → PaddleOCR-VL-0.9B (NaViT-style dynamic-resolution encoder + ERNIE-4.5-0.3B). 1.5 (Jan 29, 2026) added seal recognition + text spotting + distortion robustness (Real5-OmniDocBench). 1.6 (May 28, 2026) added region-aware data optimization + RL post-training; architecture unchanged from 1.5 (zero-cost migration).
- **Benchmarks (vendor):** v1 = 92.56/92.86, 1.5 = 94.5, 1.6 = 96.33 (v1.6). Text edit 0.033–0.035, formula CDM up to 97.5, Table TEDS ~94.8. New SOTA on Real5-OmniDocBench (in-the-wild distortions) at ~92.05. 109 languages.
- **Throughput/hardware (vendor cross-GPU, 72 DPI, PaddleOCR-VL-1.5):** A100 ~2.0 pages/s (FastDeploy) / 1.38 (vLLM) / 1.23 (SGLang); H800 2.43/1.78; A10 ~1.15/1.09/0.90; RTX 3060 ~0.54; VRAM as low as ~11–13 GB (vLLM) / ~25 GB (FastDeploy). Independent (Spheron): ~45 pages/min peak on L40S, ~60 on A100-80G, but sustained end-to-end ~half that. Runs on L4 (Google Cloud Run smallest config, L4+16 GB RAM).
- **Serving:** vLLM (official since Nov 2025), SGLang, FastDeploy, llama.cpp (merged Feb 2026, Q4_K_M ~300 MB), transformers. **Critical gotcha:** you must run the full two-stage PaddleOCR pipeline (layout + VLM), not just the VLM, or hallucination increases and paper accuracy isn't reproduced. Requires NVIDIA compute capability ≥8 (Ampere+); T4/V100 (CC 7.x) prone to OOM/timeouts and not recommended. Blackwell needs a special path.
- **Formats:** Markdown, HTML tables, JSON layout, LaTeX, charts, seals, text spotting, cross-page table merging (1.6). 109 languages — strongest multilingual coverage.
- **License:** Apache-2.0.
- **Limitations:** Two-stage CPU layout step can bottleneck GPU utilization under high concurrency; pipeline complexity vs single end-to-end models. Independent academic benchmarks rate it the most robust on vertical/multilingual/messy text among the compact models (e.g., MORE benchmark 87.96, second only to HunyuanOCR/Gemini).
- **AKS verdict:** Best for multilingual and in-the-wild (scanned/photographed/skewed) documents; solid throughput; needs Ampere+ and full-pipeline deployment.

### 5. dots.ocr (rednote-hilab/Xiaohongshu; 1.7B, renamed dots.mocr / v1.5 in 2026)
- **Architecture:** Single VLM (~1.7B LLM decoder + vision encoder); prompt-switchable layout detection + recognition. dots.mocr adds SVG parsing of charts/graphics.
- **Benchmarks:** ~88.4 OmniDocBench v1.5; olmOCR-bench ~79.1; SOTA text/table/reading-order among ~3B models. Strongest OSS specialist on independent blind voting (OCR Arena #12, ELO 1442) and Qnovi character-accuracy tests; second overall on GlotOCR Bench and strong (84.31) on the MORE multilingual benchmark.
- **Throughput/hardware:** Slowest of the group — ~0.10 images/s, ~0.35 PPS (A100), ~78.5 GB VRAM at high batch in one measurement; needs Ampere+ (A10G/L4/A100). Requires ~24 GB practical.
- **Serving:** vLLM (officially integrated since v0.11.0; `vllm/vllm-openai` Docker image; `--trust-remote-code`; model dir must have no periods). Transformers (slower, `use_hf=True`).
- **Formats:** JSON layout + bounding boxes, Markdown, HTML tables, LaTeX; 100+ languages (strong low-resource).
- **License:** MIT.
- **Limitations:** Weak on complex tables/formulas (compact model); **incomplete page transcription** — skips text-dense regions it mistakes for pictures on colorful layouts (independent French benchmark); endless repetition on ellipses/underscores; pictures not parsed; slow throughput; not optimized for high-volume PDF.
- **AKS verdict:** Best raw accuracy/reading-order for structured multilingual documents and forms where per-region JSON+boxes are needed, if throughput is secondary.

### 6. MinerU 2.5 / MinerU2.5-Pro / 3.x (OpenDataLab)
- **Architecture:** Full ingestion platform, not just a model. Decoupled two-stage 1.2B VLM (downsampled global layout → native-res crop recognition). Multiple backends: pipeline (rule+light models, PP-OCRv6, 4 GB VRAM, CPU-capable, ~86.2 OmniDocBench v1.5), vlm-engine (vLLM/SGLang/LMDeploy/MLX, 8 GB+, 90+), hybrid (native-text + VLM, low hallucination), http-client (remote VLM, 2 GB). mineru-router for multi-GPU load-balancing; async task API; streaming disk writes.
- **Benchmarks:** MinerU2.5 90.67 (v1.5)/92.98 (v1.6); MinerU2.5-Pro 95.69 (v1.6, via data engineering only, same 1.2B arch). Table TEDS +5.54 in Pro.
- **Throughput/hardware:** ~2.12 pages/s and ~2337 tokens/s on A100 (vLLM async); >10,000 tokens/s on 4090 (vlm). MLX gives 100–200% speedup on Apple Silicon. Runs on T4-class (Volta+) at 8 GB for VLM; 4 GB for pipeline.
- **Serving:** vLLM/SGLang/LMDeploy/transformers/MLX; MCP server; LangChain/Dify/FastGPT integrations; Docker; FastAPI. Dynamic repetition suppression (per-layout frequency/presence penalties). Supports 10+ domestic Chinese accelerators.
- **Formats:** Markdown/JSON; PDF/DOCX/PPTX/XLSX/images/web; 109 languages; cross-page table merging (in progress).
- **License:** Moved from AGPLv3 to a custom Apache-2.0-based "MinerU Open Source License" (April 2026) — removes enterprise copyleft friction.
- **Limitations:** Near-bottom on independent multi-script socOCRbench (Latin/CJK-print biased, ~0.165); collapses on non-Latin (MORE benchmark 48.85, last by a wide margin vs HunyuanOCR 92.42); pipeline complexity.
- **AKS verdict:** Best when you need a batteries-included ingestion layer (multi-format, routing, RAG integrations) rather than a single served model; excellent performance-per-parameter and CPU/low-VRAM fallback.

### 7. Other notable contenders
- **HunyuanOCR-1.5 (Tencent, ~1B):** 94.74 OmniDocBench v1.6; strong on coordinates, faithful reproduction (CHAOS-Bench 14.15 vs peers <7 — least likely to "correct" the source), text-image translation, and multilingual (MORE 92.42, top open model). vLLM (accuracy gap vs transformers being fixed). Strong long-tail/ancient-script focus.
- **olmOCR-2 (AllenAI, 7B, Apache-2.0):** English-focused PDF linearization, RLVR-trained; olmOCR-bench ~82.4; ~1.78 pages/s; needs H100-class. Batch/RAG-corpus oriented; OCR Arena #19 (ELO 1382). Nemotron Parse 2.0 (NVIDIA, 0.9B, Aug 3, 2026) is a multilingual/chart-aware alternative running on Ampere.
- **Nanonets-OCR2/OCR-3 (~3-4B, Apache-2.0):** Semantic tagging, KIE; Nanonets OCR-3 leads the (self-hosted Nanonets) olmOCR-bench leaderboard at ~87.4% — treat with caution as vendor-hosted. Independent tests rated Nanonets-OCR2 among the weakest on some sets.
- **Granite-Docling-258M (IBM, Apache-2.0):** Smallest; DocTags/Docling-native JSON; strong tables (TableFormer heritage); English-first; great for IBM Docling pipelines.
- **NVIDIA Nemotron-OCR-v2 (Apr 2026, ~84M):** Extreme speed (34.7 pages/s multilingual on A100, ~28× PaddleOCR v5), TensorRT/NIM, 6 languages; recognition-focused (not full document parsing).
- **PP-OCRv6 (Baidu, 34.5M):** Tiny CTC+NRTR recognizer; faithfully reproduces misspellings without linguistic-prior hallucination — a hallucination-defense complement to VLMs.
- **Mistral OCR 4 (managed API):** Bounding boxes, typed blocks, ~$4/1000 pages; strong printed text/tables; useful managed comparison point / fallback for hard cases, but not self-hostable.
- **Qwen3-VL / LightOnOCR-2-1B / Chandra / MonkeyOCR-Pro / FireRed-OCR / Youtu-Parsing / Qianfan-OCR:** Various niches; LightOnOCR-2 fast on H100 (~42.8 pages/s) and European-language focused; Chandra tops some olmOCR-bench aggregators (~85.9).

### Summary comparison table
| Model | Size | OmniDoc v1.6 (vendor) | Independent signal | ~VRAM (FP16) | Peak pages/s (A100) | Serving | Long-doc | License | Best for |
|---|---|---|---|---|---|---|---|---|---|
| GLM-OCR | 0.9B | ~95.2 | OCR Arena #24; olmOCR ~75.2 (weak tiny/long text) | ~3 GB | 1.86 | vLLM+MTP, SGLang, Ollama, MLX | page-by-page | MIT (+Apache layout) | Single-page KIE/receipts/seals, fast batch |
| PaddleOCR-VL-1.6 | 0.9B | 96.33 | Best on multilingual/vertical/messy | ~11–13 GB (vLLM) | ~1.4–2.0 | vLLM, SGLang, FastDeploy, llama.cpp | page-by-page + xtable merge | Apache-2.0 | Multilingual, in-the-wild scans |
| Baidu Unlimited-OCR | 3B/0.5B act | 93.92 | thin independent data | ~8–12 GB | n/a (35% faster long) | vLLM (custom img), SGLang (FA3/Hopper) | one-shot multi-page | MIT | Long multi-page contracts/books |
| DeepSeek-OCR / 2 | 3B/~0.5B act | 91.09 (v1.5) | near-bottom multi-script; ~9% catastrophic on hard scans | ~6.7 GB | ~2–3 (200k/day) | vLLM (n-gram LP), Ollama, llama.cpp | page-by-page | MIT / Apache-2.0 | Bulk clean-PDF digitization |
| dots.ocr | 1.7B | ~88.4 (v1.5) | OCR Arena #12 (best OSS); incomplete pages on color | ~24 GB | ~0.35 | vLLM 0.11+ | page-by-page | MIT | Structured forms, JSON+boxes |
| MinerU2.5-Pro | 1.2B | 95.69 | near-bottom multi-script; strong Latin/CJK | 8 GB (vlm)/4 GB (pipe) | ~2.12 | vLLM/SGLang/LMDeploy/MLX + router | xpage merge (WIP) | custom Apache-based | Full-stack multi-format ingestion |
| HunyuanOCR-1.5 | ~1B | 94.74 | most faithful (CHAOS 14.15); strong multilingual | ~3–4 GB | — | vLLM | page-by-page | open | Coordinates, faithful/ancient scripts |
| olmOCR-2 | 7B | — | olmOCR ~82.4; OCR Arena #19 | H100-class | ~1.78 | vLLM/SGLang | page-by-page | Apache-2.0 | English RAG-corpus linearization |

## Recommendations

**Overall architecture (all workloads).** Serve the VLM behind vLLM's OpenAI-compatible endpoint as a stateless GPU Deployment on a dedicated AKS GPU node pool (NVIDIA GPU Operator + device plugin), fronted by a lightweight ingestion/rasterization gateway (CPU pods) that pushes jobs onto a queue (Azure Service Bus/Redis). Autoscale GPU pods with KEDA on queue depth (not CPU/GPU-util alone), and store weights on a ReadOnlyMany PVC (Azure `premium-rwo`/Azure Files) to make scale-out fast. Disable prefix caching and the multimodal processor cache for OCR (each request has a distinct image). Set a generous `startupProbe` (vLLM cold start / CUDA-graph compile can take minutes). This decoupled pattern mirrors the public neural-maze production-OCR AKS reference.

**Stage 1 — start here (single-page invoices/receipts + KIE, and general high-throughput batch):** Deploy **GLM-OCR** with vLLM + MTP speculative decoding on **L4 or A10 (24 GB)** nodes. It is the fastest small model (1.86 pages/s), fits in ~3 GB (so you can pack multiple replicas or use MIG on H100), is MIT-licensed, and has the best KIE/schema and seal/receipt story. Use its JSON-schema KIE for structured extraction. Benchmark against **PaddleOCR-VL-1.6** on your own documents in parallel.
- *Threshold to switch:* if your corpus is heavily multilingual, scanned/photographed/skewed, contains vertical or mid-resource-script text, or if GLM-OCR's tiny/long-text weakness shows up → move to **PaddleOCR-VL-1.6** (full two-stage pipeline, Ampere+ GPU).

**Stage 2 — long multi-page contracts/reports/books:** Deploy **Baidu Unlimited-OCR** for one-shot long-horizon parsing to avoid page-splitting/stitching errors and cross-page table breaks. Provision **Hopper GPUs (H100/H200)** to use the FA3 R-SWA long-context serving path; use the dedicated vLLM image or the bundled SGLang dev-build wheel, and keep the n-gram logits processor enabled. If you cannot run Hopper, fall back to page-by-page **PaddleOCR-VL-1.6** (cross-page table merging) or the MinerU hybrid backend.

**Stage 3 — high-throughput bulk digitization of clean digital PDFs:** **DeepSeek-OCR** (or OCR-2) on A100/L40S maximizes pages-per-dollar via its MoE + optical compression (~200k pages/day/A100). Mandatory: n-gram logits processor + a repetition/looping detector with automatic retry (independent tests show ~9% catastrophic failures on hard scans; budget for it). For mixed/messy corpora prefer MinerU or PaddleOCR-VL instead.

**Stage 4 — formula/scientific and complex tables:** GLM-OCR (formula CDM 93.9, UniMERNet 96.5), PaddleOCR-VL-1.6 (CDM up to 97.5), or MinerU2.5-Pro (dense-formula CDM 97.29). All are strong; pick based on your Stage-1/2 choice to minimize model sprawl.

**Cross-cutting guardrails.**
- Always add a repetition/hallucination detector (n-gram loop detection, output-length ceilings, blank-page handling) — this is the #1 production failure across all VLM-OCR (LlamaIndex traced LlamaParse outages to exactly this).
- Validate on YOUR document distribution: pull 20–50 representative pages, measure text accuracy, table TEDS, and page-boundary integrity end-to-end. Do not trust OmniDocBench deltas — it is saturated and Latin/CJK-print-biased.
- Consider PP-OCRv6 or a rules-based checker as a hallucination cross-check for high-stakes fields (finance/legal/medical), since VLMs "correct" text using language priors.
- Keep a managed API (Mistral OCR 4 or Gemini 3 Flash/Pro) as an overflow/hard-case fallback if compliance permits — proprietary models still lead independent multi-script benchmarks by a wide margin.

## Caveats
- **Vendor vs. independent scores.** OmniDocBench v1.5/v1.6 scores for GLM-OCR, PaddleOCR-VL, DeepSeek-OCR, Unlimited-OCR and MinerU are vendor-reported (models often self-evaluated while competitors are pulled from the OmniDocBench repo). Independent reorderings (socOCRbench, OCR Arena, olmOCR-bench, GlotOCR Bench, MORE, and several arXiv multilingual benchmarks) diverge sharply, especially on non-Latin/handwritten/degraded text. The benchmark is saturated per LlamaIndex.
- **Throughput figures are best-case.** Vendor pages/s numbers are single-image or large-batch peaks with vLLM startup excluded; sustained end-to-end throughput (including rasterization/I/O) is roughly half (per Spheron). Numbers vary with DPI, resolution mode, batch size, and GPU. No published T4/L4 pages/s figures exist for most models; A10 is the lowest reliable data point (PaddleOCR-VL-1.5 ~1.1 pages/s).
- **Speculative/future items flagged.** R-SWA generalization to ASR/translation is stated as future work, not shipped. Unlimited-OCR's roadmap (128K context, auto page-turning) is a paper outlook. Some benchmark aggregators inflate scores using quantized variants.
- **Licensing wrinkles.** DeepSeek-OCR's MIT code pulls AGPL PyMuPDF (redistribution question raised in GitHub Issue #223); GLM-OCR is MIT but bundles Apache-2.0 PP-DocLayout-V3; MinerU moved off AGPLv3 to a custom Apache-based license in April 2026. Verify the exact weights license on each Hugging Face repo before shipping.
- **Newer releases since June 2026** (PaddleOCR-VL-1.6 May 28; MinerU2.5-Pro-2605 May 21; HunyuanOCR-1.5; NVIDIA Nemotron Parse 2.0 Aug 3) continue to shift the leaderboard monthly — re-validate quarterly.
- **Coverage gap:** Independent production evaluation of Baidu Unlimited-OCR is thin (released June 2026); most confidence there rests on vendor claims plus its DeepSeek-OCR lineage — pilot it on your own long documents before committing.