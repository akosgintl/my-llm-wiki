# Deep Technical Review: Open-Source OCR / Visual Document Understanding Models — Missed Details & Pipeline Implications

## TL;DR
- **The single most consequential finding for this Hungarian-heavy, GLM-OCR-fixed pipeline: GLM-OCR officially claims only 8 languages (Chinese, English, French, Spanish, Russian, German, Japanese, Korean) — Hungarian is NOT among them**, whereas **PaddleOCR-VL explicitly lists Hungarian in its 109-language Latin set (but does not support fine-tuning)** and **Qwen3-VL's technical report claims 39 OCR languages with >70% accuracy on 32 of them**. Hungarian coverage is therefore a hard constraint that must drive model selection and LoRA data design — GLM-OCR LoRA fine-tuning is effectively mandatory, not optional.
- **Several pipeline-critical facts are underdocumented:** GLM-OCR's MTP head survival under LoRA-merge is undocumented (an open risk for the ~50% speculative-decoding throughput gain); DeepSeek-OCR ships an AGPL-3.0 PyMuPDF dependency that legally contaminates its MIT claim; and the DeepSeek-OCR critique paper shows accuracy behavior that directly validates the need for the confidence-scoring stack.
- **Exact resolution/token budgets, prompt strings, repetition-control flags, licensing, and export formats** (notably PP-DocLayoutV3 ONNX for a CPU layout tier) are now pinned down per model below, with the highest-impact anti-hallucination signal being HunyuanOCR-1.5's CHAOS-Bench faithfulness score (14.15 vs 5–6 for peers).

## Key Findings
1. **Hungarian coverage is bimodal.** PaddleOCR-VL (109→111 langs, Hungarian explicitly listed) and Qwen3-VL (39 OCR langs) name Latin/Hungarian coverage; GLM-OCR (8 langs) and DeepSeek-OCR do not; LightOnOCR-2 is Latin-script but French-weighted with no Hungarian confirmation.
2. **GLM-OCR MTP** is trained to predict 10 tokens/step and generates 5.2 tokens/step at inference (~50% throughput gain), reusable as a vLLM speculative draft — but LoRA-merge preservation is undocumented.
3. **DeepSeek-OCR resolution modes are exact and small**: 64/100/256/400 vision tokens; Gundam = n×100+256; ~97% decode precision at <10× compression, ~60% at 20×.
4. **PaddleOCR-VL fine-tuning is officially unsupported** (docs FAQ), contradicting any assumption that it is swappable-and-trainable.
5. **PP-DocLayoutV3 has validated community ONNX exports** enabling a Paddle-free CPU layout tier — important for the 24GB/CPU fleet.

## Per-Model Review

### 1. GLM-OCR (Zhipu/Z.ai) — the fixed recognition model
- **arXiv:** https://arxiv.org/abs/2603.10910 (HTML: https://arxiv.org/html/2603.10910v2) · **GitHub:** https://github.com/zai-org/GLM-OCR · **HF:** https://huggingface.co/zai-org/GLM-OCR
- **Study already covers:** 0.9B (0.4B CogViT + 0.5B GLM decoder), PP-DocLayout-V3 two-stage pipeline, MTP, GRPO, 94.62 OmniDocBench v1.5, 1.86 pages/sec, Apache-2.0 code / MIT model.
- **MISSED technical details:**
  - **GRPO reward (Table 2):** Text = Normalized Edit Distance + repetition penalty; Formula = CDM score + structural-validity check; Table = TEDS + tag-closure verification + structural parsing; KIE = field-level F1 + JSON-parse validation + missing/duplicate-field penalty; plus a GLOBAL repetition-ratio penalty and malformed-structure penalty. Crucially, **no closed-form equation is published** — only this design table. Rollouts are stratified by difficulty into a graded optimization set.
  - **MTP:** Verbatim from the paper — "GLM-OCR is trained to predict ten tokens per step and generates 5.2 tokens per decoding step on average at inference time, bringing approximately 50% throughput improvement." Implemented as k shared-parameter auxiliary heads modeling different future offsets; MTP kept enabled through SFT. Reusable as a vLLM speculative draft: `--speculative-config '{"method":"mtp","num_speculative_tokens":3}'`; SGLang equivalent: `--speculative-algorithm NEXTN --speculative-num-steps 3 --speculative-eagle-topk 1 --speculative-num-draft-tokens 4` with `SGLANG_ENABLE_SPEC_V2=1`.
  - **LoRA + MTP — OPEN RISK:** The LLaMA-Factory finetune README (added 2026-02-12) never mentions the MTP/NextN head. LoRA config: `lora_rank: 8, lora_target: all`; full-SFT freezes vision tower + projector and trains the LM only. Whether `llamafactory-cli export` merge yields a functional MTP head is undocumented in repo, paper, or any issue — must be validated empirically before relying on speculative decoding post-fine-tune.
  - **Exact prompt strings:** `{"text":"Text Recognition:","formula":"Formula Recognition:","table":"Table Recognition:"}`, with the `<image>` special token placed BEFORE the text prompt (e.g. `<image>Text Recognition:`). Custom prompts are allowed (the training data includes `<image>Code Generation:`). KIE has NO canonical string — it is a free-form strict-JSON-schema instruction and the output must strictly adhere to the schema.
  - **Sub-metrics (from paper, benchmarking context):** Overall 94.62 on OmniDocBench v1.5 (vs PaddleOCR-VL-1.5 94.50, MinerU2.5 90.67, Qwen3-VL-235B 89.15, Gemini-3 Pro 90.33); Table_TEDS 93.96, Formula_CDM 93.90.
  - **Languages:** Paper footnote 2 lists exactly 8 languages (zh, en, fr, es, ru, de, ja, ko). **Hungarian NOT listed.** "100+ languages" claims are unverified marketing not present in the paper/README/model card. Limitations note degradation on underrepresented languages.
  - **CogViT/connector:** encoder trained on tens of billions of image-text pairs via MIM+CLIP + distillation; connector described only qualitatively as "efficient token downsampling" — **no numeric downsampling ratio disclosed.**
  - **SDK config convention:** priority chain = constructor kwargs > os.environ > .env > config.yaml > defaults. Env vars prefixed `GLMOCR_` (API key = `ZHIPU_API_KEY`). Keys include `pipeline.maas.enabled`, `pipeline.ocr_api.api_host/api_port` (default 8080), `page_loader.min_pixels: 12544 / max_pixels: 71372800`, `result_formatter.output_format: both`, `layout.device`. Four server modes: MaaS cloud, self-host vLLM/SGLang, Ollama/MLX, SDK-server+client (Flask on :5002, endpoint `/glmocr/parse`). Region bbox uses normalized 0–1000 scale. Layout can run on CPU (`--layout-device cpu`) to keep the GPU free for OCR.
- **Pipeline implications:** Hungarian ő/ű is out-of-distribution for GLM-OCR's 8-language claim → LoRA fine-tuning on a Hungarian golden set is essentially mandatory. Validate MTP after merge or plan to serve base+adapter unmerged (losing the ~50% throughput). The `<image>`-before-prompt ordering and byte-exact prompt strings must be preserved in the ResultFormatter path. KIE's strict-JSON path integrates cleanly with the value-in-OCR grounding confidence stack.

### 2. PaddleOCR-VL (v1 / 1.5 / 1.6) + PaddleOCR 3.0
- **arXiv:** v1 https://arxiv.org/abs/2510.14528 · 1.5 https://arxiv.org/abs/2601.21957 · 1.6 https://arxiv.org/abs/2606.03264 · PaddleOCR 3.0 https://arxiv.org/abs/2507.05595 · **GitHub:** https://github.com/PaddlePaddle/PaddleOCR
- **Study already covers:** 0.9B NaViT-style dynamic-resolution encoder + ERNIE-4.5-0.3B, 109 languages, two-stage (PP-DocLayoutV2 layout → VLM recognition), SOTA OmniDocBench.
- **MISSED details:**
  - **Hungarian CONFIRMED** in the Latin category of the 109-language appendix (Table A1: "…Croatian, Uzbek, **Hungarian**, Serbian (Latin)…"); a PaddlePaddle maintainer confirmed the full list is in the technical report appendix. v1.5 adds Tibetan + Bengali → **111 languages.**
  - **Fine-tuning OFFICIALLY UNSUPPORTED:** docs FAQ verbatim — "Currently, we do not support fine-tuning of the model, but it is a high-priority feature and will be released soon." This CONTRADICTS treating PaddleOCR-VL as a trainable swappable model.
  - **Chart recognition default-OFF:** `use_chart_recognition=True` must be set manually.
  - **v1.6:** region-aware "under-optimized region" (boundary-fragile / coverage-sparse / unreliable-supervision) data engine + progressive CPT→SFT→RL post-training; 94.93→**96.33** on OmniDocBench v1.6; architecture unchanged from v1.5 (zero-cost plug-and-play migration); strong gains on tables/charts/formulas/seals/real-world images.
  - **Reading order:** PP-DocLayoutV2 localizes semantic regions and predicts reading order (Global-Pointer mechanism). Table target format is OTSL, not HTML.
  - **Deployment:** vLLM/SGLang/FastDeploy; OpenVINO path via `paddleocr_vl_ov`.
- **Pipeline implications:** PaddleOCR-VL is the strongest Hungarian-native option, but the no-fine-tuning limitation means it cannot be adapted to format-native output targets the way GLM-OCR can — deploy it as a benchmark oracle / second-opinion signal rather than the trainable primary.

### 3. DeepSeek-OCR & DeepSeek-OCR 2
- **arXiv:** v1 https://arxiv.org/abs/2510.18234 · v2 https://arxiv.org/abs/2601.20552 · critique https://arxiv.org/abs/2601.03714 · **GitHub:** https://github.com/deepseek-ai/DeepSeek-OCR , https://github.com/deepseek-ai/DeepSeek-OCR-2
- **Study already covers:** DeepEncoder (SAM-base patch-16 window attention + CLIP-large global, 16× conv downsample) + DeepSeek-3B-MoE decoder (570M active, 6 routed + 2 shared experts), optical-compression thesis.
- **MISSED details:**
  - **Exact resolution modes:** Tiny 512² = 64 tokens; Small 640² = 100; Base 1024² = 256; Large 1280² = 400. Gundam = n×640² tiles + 1×1024² global = **n×100+256 tokens** (n=0 if both dims <640). Gundam-Master = 1024² local + 1280² global (obtained by continued training). Base/Large pad to aspect ratio → valid tokens < allocated after padding. A 1024² image = 4096 patch tokens → compressed 16× → 256.
  - **Compression ablation (Fox benchmark):** "~97% decoding precision at <10× compression … ~90% at 10-12× … ~60% at 20× compression" — verbatim from the paper. This is the quantitative backbone of the confidence stack's risk model.
  - **Prompt grammar (newline-sensitive):** `<image>\n<|grounding|>Convert the document to markdown.` (document); `<image>\n<|grounding|>OCR this image.`; `<image>\nFree OCR.` (no layout); `<image>\nParse the figure.`; `<image>\nLocate <|ref|>xxxx<|/ref|> in the image.` (grounded).
  - **DeepSeek-OCR 2 / DeepEncoder V2:** dynamically reorders visual tokens via causal reasoning; local views share **144 query embeddings**; total reordered tokens = **k×144+256, range [256, 1120]**; k crops 0–6 (no crop if both dims <768). Max 1120 < DeepSeek-OCR's 1156 (Gundam) and matches Gemini-3-Pro's budget. Attention mask = bidirectional (vision, ViT-like) concatenated with causal-triangular (flow tokens, decoder-style); only the causal flow tokens (latter half of encoder outputs) are fed to the LLM. Still a DeepSeek-3B-MoE decoder.
  - **Critique paper (arXiv 2601.03714):** under sentence/word-level semantic corruption, DeepSeek-OCR accuracy plummets from ~90% to ~20%; lower visual-token counts correlate with MORE reliance on language priors and higher hallucination; traditional pipeline OCR methods are significantly more robust than end-to-end methods (13 baselines compared).
  - **AGPL contamination:** Issue #223 (opened Nov 5, 2025) — "PyMuPDF is licensed under the AGPL-3.0 or a commercial license. Thus, this project becomes a derivative work and would have to be licensed under the same terms (AGPL-3.0) due to the strong copyleft." The MIT claim is disputed.
  - **Unsloth:** the stock checkpoint is not runnable on latest transformers; Unsloth ships a modified `unsloth/DeepSeek-OCR` (incorporating Stranger Vision HF changes); demoed 88.26% CER improvement on Persian (149.07%→60.81% CER, 60 steps, batch 8). vLLM 0.11.0 (v1 engine) needs a custom `NoRepeatNGramAdaptor` logits processor with `ngram_size=30`, `temperature=0.0`, `skip_special_tokens=False`.
- **Pipeline implications:** The AGPL PyMuPDF dependency is a genuine legal blocker for a commercial pipeline — isolate or replace `fitz` if any DeepSeek-derived model enters production. The critique's finding directly validates the value-in-OCR grounding + Hungarian checksum validators: low-token modes will hallucinate plausible Hungarian words, so confidence calibration must penalize low-visual-evidence outputs.

### 4. Baidu Unlimited-OCR
- **arXiv:** https://arxiv.org/abs/2606.23050 · **GitHub:** https://github.com/baidu/Unlimited-OCR
- **MISSED details:** R-SWA replaces ALL decoder attention layers. Each generated token attends to (a) all static reference tokens (visual + prompt, length m fixed at inference start) and (b) a sliding window of the last **n=128** output tokens; KV cache is bounded at **m+n** (older tokens evicted) — cache size C(T) = Lm + min(n,T) ≤ Lm + n. Visual tokens are excluded from state transitions to preserve fidelity and avoid progressive blurring. Continue-trained from the DeepSeek-OCR checkpoint (decoder-only, DeepEncoder frozen), ~4000 steps, on 8×16 A800; ~2M samples, 9:1 single:multi-page concatenation. `max_length=32768` (prompt+output combined). Prompt must start with `<image>`; `<|ref|>/<|det|>` tokens stripped from output. MoE 3B total / 0.5B active. 93.23 OmniDocBench v1.5 (+6.22 over baseline), 93.92 v1.6. Multi-page batching limited to base mode. MIT license; roadmap targets 128K context + prefill pool.
- **Pipeline implications:** Strong for multi-page Hungarian docs where split/merge is painful; the constant KV cache is attractive for MIG-partitioned H100s and 24GB GPUs. But it inherits DeepSeek-OCR's language-prior hallucination risk and lacks Hungarian confirmation.

### 5. dots.ocr / dots.mocr
- **GitHub:** https://github.com/rednote-hilab/dots.ocr , https://github.com/rednote-hilab/dots.mocr · **HF:** https://huggingface.co/rednote-hilab/dots.ocr
- **MISSED details:** 1.7B built on the **Qwen2.5-1.5B** foundation; handles up to **11,289,600 pixels** (downsample or raise DPI to 200 above that). Prompt-switchable modes incl. `prompt_layout_all_en` (detect+recognize) and `prompt_layout_only_en` (**layout-only — usable as a second-opinion signal**). JSON output carries bbox + category + text. Repetition triggers: continuous special characters (ellipses `...`, underscores `___`). **Pictures NOT parsed** (gap for embedded infographics). Model directory must contain **NO periods** (use `DotsOCR`, not `dots.ocr`) — a temporary Transformers-integration workaround. Officially integrated in **vLLM ≥0.11.0**; published evals used vLLM 0.9.1. dots.mocr / dots.mocr-svg add SVG chart parsing. Default `num_thread=64`; supports temperature/top_p/max_completion_tokens.
- **Pipeline implications:** layout-only mode is a cheap independent layout cross-check; the no-periods dir gotcha and special-char repetition triggers must be guarded in the ResultFormatter and prompt-sanitization path.

### 6. MinerU 2.5 / 2.5-Pro
- **arXiv:** https://arxiv.org/abs/2509.22186 · Pro https://arxiv.org/abs/2604.04771 · **GitHub:** https://github.com/opendatalab/MinerU
- **MISSED details:** 1.2B = **NaViT-675M vision + Qwen2-0.5B decoder.** Two-stage coarse-to-fine: stage-1 layout on a DOWNSAMPLED image, stage-2 recognition on native-resolution crops — avoids the O(N²) visual-token blowup of end-to-end native-res models. Preserves headers/footers/page-numbers with a refined, standardized labeling schema (lists, references, code blocks). **Dynamic per-element repetition suppression:** frequency + presence penalties adjusted in stage-2 to avoid killing legitimate repetitive structures (tables/equations). Backends: `transformers`, `vllm-engine`, `vllm-async-engine` (2.12 fps on one A100), `http-client`, plus a mineru-router; helper package `mineru-vl-utils`. MinerU2.5-Pro: identical architecture (initialized from MinerU2.5 Stage-0), data-engineering only (DDAS + CMCV + Judge-and-Refine; <10M→65.5M pages); 92.98→**95.69** on OmniDocBench v1.6.
- **Pipeline implications:** The coarse-to-fine design mirrors the pipeline's own layout-then-recognize split; MinerU's dynamic per-element penalty is a proven recipe to port into the vLLM sampling parameters for Hungarian tables/formulas.

### 7. HunyuanOCR-1.5 (Tencent)
- **arXiv:** https://arxiv.org/abs/2607.04884 (v1.0: https://arxiv.org/abs/2511.19575) · **GitHub:** https://github.com/Tencent-Hunyuan/HunyuanOCR
- **MISSED details:** ~1B end-to-end; adds **DFlash speculative decoding** + Agentic Data Flow over v1.0 (backbone unchanged). **CHAOS-Bench** (character-level hallucination / faithfulness under visual-vs-prior conflict) page-avg recall: **HunyuanOCR-1.5 = 14.15** vs DeepSeek-OCR 2 = 6.33, MinerU2.5-Pro = 6.33, PaddleOCR-VL-1.6 = 5.95, GLM-OCR = 5.75, dots.ocr = 3.02 — dramatically higher faithfulness. OmniDocBench v1.6 Overall = **94.74.** Unifies document parsing, text spotting, IE, text-image translation (built-in), multi-image document QA. **MORE** multilingual bench spans 149 low-resource languages (Hungarian membership not individually confirmed). llama.cpp PC deployment. License: **NOASSERTION** (needs review before commercial use).
- **Pipeline implications:** HunyuanOCR-1.5's CHAOS-Bench dominance directly attacks the "canonical failure mode" — hallucinating plausible Hungarian words when visual evidence is weak. It is the most faithfulness-robust candidate and should be benchmarked head-to-head against GLM-OCR on the golden set.

### 8. olmOCR-2 (AllenAI)
- **arXiv:** https://arxiv.org/abs/2510.19817 · **GitHub:** https://github.com/allenai/olmocr · **blog:** https://allenai.org/blog/olmocr-2
- **MISSED details:** olmOCR-2-7B-1025 built on **Qwen2.5-VL-7B**, fine-tuned on olmOCR-mix-1025 (270k pages + 20k added hard handwritten/typewritten). **RLVR via GRPO with binary unit-test rewards** aggregated to a page-level pass rate (e.g. 4/6 = 0.67). Synthetic pipeline renders any doc to clean HTML then extracts verifiable unit tests; math-heavy arXiv pages used for hard cases; teacher = claude-sonnet for HTML generation. Dynamic temperature scaling for retry. **82.4 on olmOCR-Bench** (English-only; +14.2 over prior version), beating Marker (76.1) and MinerU (75.8). Fully open data/model/code.
- **Pipeline implications:** English-only bench limits direct Hungarian relevance, but the unit-test-reward + retry/filter heuristics are a portable blueprint for designing the Hungarian checksum-validator reward.

### 9. LightOnOCR-2-1B
- **arXiv:** https://arxiv.org/abs/2601.14251 · **HF:** https://huggingface.co/lightonai/LightOnOCR-2-1B · **blog:** https://huggingface.co/blog/lightonai/lightonocr-2
- **MISSED details:** 1B end-to-end; pretraining scaled 17M→43M pages; teacher = **Qwen3-VL-235B-A22B-Instruct**; max longest-edge 1024→**1540px**; RLVR + task-arithmetic ("soup") merging. **83.2 ± 0.9 on olmOCR-Bench**; 5.71 pages/s on one H100 (~493k pages/day, <$0.01/1k pages). Variants: OCR-only, `-bbox`, `-bbox-soup`, plus base checkpoints for RLVR. **European/Latin-script focus but explicitly French-weighted; HUNGARIAN NOT confirmed** — §6 states multilingual performance outside European/Latin scripts is not fully supported and non-Latin scripts degrade; 3% blank-page examples included to prevent hallucination loops. Apache-2.0.
- **Pipeline implications:** Fast and Latin-script-capable, but no Hungarian evidence and a French skew — treat as a throughput baseline and empirically verify ő/ű before trusting.

### 10. Qwen3-VL & Qwen3.5
- **Qwen3-VL arXiv:** https://arxiv.org/abs/2511.21631 · **GitHub:** https://github.com/QwenLM/Qwen3-VL · **Qwen3.5 HF:** https://huggingface.co/Qwen/Qwen3.5-4B
- **MISSED details:**
  - **OCR languages: the technical report claims 39** — verbatim: "Qwen3-VL … supporting 39 languages with over 70% accuracy on 32 of them," backed by "30M multilingual OCR samples across 39 languages." The GitHub README says "32 languages (up from 10)" while some HF/LM-Studio cards say "up from 19" — a genuine marketing inconsistency; cite the paper's 39 (32 >70%). Hungarian membership in the 39-language OCR set is NOT individually confirmed (the Qwen3.5-Omni Fleurs top-59 DOES list Hungarian, but that is speech, not OCR).
  - **Visual token budget:** 256–1280 tokens at 32× compression, dimensions as multiples of 32; recaptioning with Qwen2.5-VL-32B; 7M PDFs referenced in the data pipeline.
  - **Architecture:** Interleaved-MRoPE (full-frequency time/width/height), DeepStack multi-level ViT fusion, stage-0 merger-only training, Text–Timestamp alignment; 256K native context (→1M). QwenVL-HTML output format for structured pages.
  - **Qwen3.5:** Gated DeltaNet hybrid attention (~3:1 GDN:full, i.e. 75% linear-attention layers) + sparse MoE; 262K native context (→1M+). **201 languages/dialects (up from Qwen3's 119) is a general text claim, NOT OCR-specific.** vLLM 0.17+ added GDN kernel support + a hybrid KV-cache manager (distinct code path; may need `--trust-remote-code`); MTP supported in vLLM for the Qwen3-Next lineage.
- **Pipeline implications:** The neural-maze reference course uses Qwen3.5-4B, but its OCR Hungarian quality is unverified and the 201-language headline does NOT imply OCR coverage. The GDN hybrid attention changes vLLM KV-cache math on MIG partitions (linear-attention layers have a different KV footprint) and must be capacity-tested before committing MIG slice sizes.

### 11. Adjacent / Implementation-Relevant
- **PP-DocLayoutV3** (DETR + PPHGNetV2-L backbone, 25 categories incl. a reading-order prediction branch, instance segmentation for skew/curve robustness): community ONNX export `alex-dinh/PP-DocLayoutV3-ONNX` (via Paddle2ONNX PR #1619) and `AlexTransformer/PP-DocLayoutV3-onnx` (MIT, validated against the Paddle native pipeline on 1355 images). **Critical gotcha:** the ONNX model expects **mean=[0,0,0], std=[1,1,1] (rescale by 1/255 only), NOT ImageNet normalization** — wrong norm drops detections from 13→12. Native torch source hardcodes batch=1 (blocks naive ONNX batching unless patched). HF `PaddlePaddle/PP-DocLayoutV3_safetensors` supports batched inference via the Transformers `object-detection` pipeline. OpenVINO path via `paddleocr_vl_ov` (add_layout branch).
- **OmniDocBench** (arXiv 2412.07626): the dominant page-level benchmark; v1.5/v1.6 corrected element-matching bias via Multi-Granularity Adaptive Matching and added a Hard subset (Base/Hard/Full tiers). Composition skews academic/Chinese-English — a bias to note when reading Hungarian-relevant claims.
- **Qianfan-OCR** (arXiv 2603.13398, Table 6), **ConfBench** (arXiv 2608.01792, workflow appendix A.5), **EXTRACTCONF** (arXiv 2606.24420, 40-feature list): confidence/extraction-eval references for the scoring stack.

## Ecosystem Repos — Judged
- **neural-maze/production-ocr-course** — https://github.com/neural-maze/production-ocr-course — Rust(Axum)+vLLM+Redis+KEDA AKS reference using the GLM-OCR SDK + PP-DocLayoutV3 + Qwen3.5-4B, /dev/shm zero-copy handoff, APIM security, scale-to-zero (asymmetric T4-layout / A100-inference node pools). **Highest value** — near-exact topological match to this pipeline. Caveat: it is free course material, not battle-tested production; validate the manifests before trusting.
- **vLLM Recipes — GLM-OCR** (docs.vllm.ai/projects/recipes) — canonical MTP speculative-decoding serving flags; authoritative. Use.
- **Unsloth** (unslothai/unsloth + `unsloth/DeepSeek-OCR`) — the only turnkey DeepSeek-OCR fine-tuning path; requires their modified checkpoint. Use for DeepSeek; not applicable to GLM-OCR.
- **GLM-OCR examples/finetune (LLaMA-Factory)** — official LoRA recipe (freezes vision + projector). Use, but validate MTP post-merge.
- **OmniDocBench repo / olmOCR-bench** — the two most credible eval harnesses; olmOCR-bench is English-only (limited Hungarian value).
- **andynoodles/PPDocLayout-V3-Benchmark** + **AlexTransformer/alex-dinh PP-DocLayoutV3-onnx** — for the CPU layout tier; validated, MIT. Use.
- **zhaohb/paddleocr_vl_ov** — OpenVINO PaddleOCR-VL pipeline; useful for CPU/Intel tiers and as a Paddle-free reference.
- **Skip:** generic "awesome-OCR" lists, one-off Gradio demo Spaces, and aggregator "DeepWiki" pages — low signal for production and often agent-scaffolded.

## Consolidated Link Index (all arXiv + GitHub URLs)
| Model / Item | arXiv | GitHub / Primary |
|---|---|---|
| GLM-OCR | 2603.10910 | github.com/zai-org/GLM-OCR |
| PaddleOCR-VL v1 | 2510.14528 | github.com/PaddlePaddle/PaddleOCR |
| PaddleOCR-VL 1.5 | 2601.21957 | (same) |
| PaddleOCR-VL 1.6 | 2606.03264 | (same); hf.co/PaddlePaddle/PaddleOCR-VL-1.6 |
| PaddleOCR 3.0 | 2507.05595 | (same) |
| DeepSeek-OCR | 2510.18234 | github.com/deepseek-ai/DeepSeek-OCR |
| DeepSeek-OCR 2 | 2601.20552 | github.com/deepseek-ai/DeepSeek-OCR-2 |
| DeepSeek-OCR critique | 2601.03714 | — |
| Unlimited-OCR | 2606.23050 | github.com/baidu/Unlimited-OCR |
| dots.ocr / dots.mocr | — | github.com/rednote-hilab/dots.ocr ; /dots.mocr |
| MinerU 2.5 | 2509.22186 | github.com/opendatalab/MinerU |
| MinerU 2.5-Pro | 2604.04771 | (same) |
| HunyuanOCR-1.5 | 2607.04884 | github.com/Tencent-Hunyuan/HunyuanOCR |
| HunyuanOCR 1.0 | 2511.19575 | (same) |
| olmOCR-2 | 2510.19817 | github.com/allenai/olmocr |
| LightOnOCR-2 | 2601.14251 | hf.co/lightonai/LightOnOCR-2-1B |
| Qwen3-VL | 2511.21631 | github.com/QwenLM/Qwen3-VL |
| Qwen3.5 | — | hf.co/Qwen/Qwen3.5-4B |
| PP-DocLayoutV3 | — | hf.co/PaddlePaddle/PP-DocLayoutV3 ; alex-dinh/PP-DocLayoutV3-ONNX |
| OmniDocBench | 2412.07626 | — |
| Qianfan-OCR | 2603.13398 | — |
| ConfBench | 2608.01792 | — |
| EXTRACTCONF | 2606.24420 | — |
| production-ocr-course | — | github.com/neural-maze/production-ocr-course |

## Top 10 Missed Details (ranked by pipeline impact)
1. **GLM-OCR officially supports only 8 languages — Hungarian NOT among them** (LoRA fine-tuning becomes mandatory, not optional).
2. **GLM-OCR MTP-after-LoRA-merge is undocumented** — the ~50% speculative-decoding throughput gain is at risk and must be validated empirically.
3. **PaddleOCR-VL fine-tuning is officially unsupported** (docs FAQ) — it cannot be the trainable primary.
4. **DeepSeek-OCR's AGPL-3.0 PyMuPDF dependency contaminates its MIT claim** (Issue #223) — a commercial-licensing blocker.
5. **DeepSeek-OCR critique:** ~90%→~20% under semantic corruption; fewer visual tokens → more hallucination — validates the grounding/checksum confidence stack.
6. **PP-DocLayoutV3 ONNX export exists + the mean=0/std=1 norm gotcha** — enables a Paddle-free CPU layout tier for the 24GB fleet.
7. **HunyuanOCR-1.5 CHAOS-Bench faithfulness = 14.15 vs 5–6 for peers** — the strongest anti-hallucination candidate for Hungarian.
8. **Exact GLM-OCR GRPO reward table** (NED / CDM / TEDS / field-F1 + tag-closure + global repetition/malformed-structure penalties).
9. **DeepSeek-OCR / Unlimited-OCR exact token budgets + repetition-control flags** (Gundam n×100+256; R-SWA n=128, KV=m+n; vLLM `ngram_size=30`).
10. **Qwen3.5 Gated DeltaNet hybrid attention changes vLLM KV-cache math on MIG partitions** — must be capacity-tested before fixing MIG slice sizes.

## Contradictions vs the Study's Current Claims
- **PaddleOCR-VL fine-tuning:** if the study assumes it is trainable/swappable, the docs explicitly say fine-tuning is unsupported (planned but not released).
- **Hungarian language-list membership:** confirmed only for PaddleOCR-VL (109/111); **NOT confirmed and likely absent for GLM-OCR (8 langs), DeepSeek-OCR, and LightOnOCR-2**; unconfirmed for Qwen3-VL's OCR-39 set and HunyuanOCR's MORE-149.
- **Qwen3-VL language-count marketing inconsistency:** "32 (up from 10)" (README) vs "32 (up from 19)" (some HF cards) vs the paper's **39 OCR languages (32 >70%)** — cite the paper's figure.
- **GLM-OCR "100+ languages":** unverified marketing; primary sources (paper footnote 2, HF card) say 8.

## Recommendations
1. **Immediate (weeks 0–2):** Treat Hungarian as unsupported by GLM-OCR out-of-the-box — proceed with the planned Unsloth/LLaMA-Factory LoRA (frozen vision encoder, format-native targets) on a Hungarian golden set with dense ő/ű coverage. In parallel, stand up **PaddleOCR-VL-1.6 and HunyuanOCR-1.5 as read-only oracles / second-opinion signals** on the same golden set.
2. **De-risk MTP (before production serving):** Empirically verify the merged LoRA checkpoint still exposes a functional MTP head under vLLM `--speculative-config method mtp`. If it does not, either serve base+adapter unmerged or budget for the ~50% throughput loss.
3. **Legal (before any DeepSeek-derived model ships):** Isolate or replace the PyMuPDF (`fitz`) dependency; review HunyuanOCR-1.5's NOASSERTION license with counsel; confirm the transitive PP-DocLayoutV3 Apache-2.0 obligation is documented.
4. **CPU/24GB layout tier:** Adopt the validated PP-DocLayoutV3 ONNX export with the correct mean=0/std=1 normalization (not ImageNet), keeping layout off the GPU (`--layout-device cpu`) to free MIG slices for recognition.
5. **Confidence stack:** Weight low-visual-token outputs as higher hallucination risk (per the DeepSeek-OCR critique), and add CHAOS-Bench-style character-perturbation tests to the golden-set calibration so the logprob + grounding + checksum ensemble is tuned against faithfulness failures, not just clean-text accuracy.
- **Thresholds that change these:** If GLM-OCR LoRA fails to reach Hungarian CER/edit-distance parity with PaddleOCR-VL-1.6 on the golden set (target: within ~1–2 CER points), switch the primary recognition model to **HunyuanOCR-1.5** (best faithfulness) or fall back to **PaddleOCR-VL-1.6** as a frozen non-trainable oracle. If MTP does not survive merge and throughput drops below the 1.86 pages/sec service target under load, either revert to unmerged adapter serving or scale replicas via KEDA.

## Caveats
- arXiv IDs dated 2026 are pre-prints; several benchmark numbers (OmniDocBench v1.5/v1.6, olmOCR-Bench, CHAOS-Bench) are self-reported by the model authors and should be independently reproduced on the Hungarian golden set.
- Hungarian OCR membership for Qwen3-VL (39-lang set) and HunyuanOCR (MORE-149) could not be individually confirmed from available sources; the Qwen Fleurs Hungarian listing is speech, not OCR.
- GLM-OCR's MTP-under-merge behavior and the CogViT connector downsampling ratio are genuinely undocumented (not merely unfound) — flag both as items requiring vendor confirmation or empirical measurement.
- The neural-maze course is the closest architectural reference but is educational material; treat its manifests as a starting template requiring production hardening.