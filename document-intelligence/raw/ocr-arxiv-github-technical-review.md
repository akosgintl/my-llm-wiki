# arXiv & GitHub Technical Review — Missed Details That Matter
*Per-model review of the primary papers and repositories for every model in the study, focused on technical information the study missed or under-specified: serving behavior, prompts/tokens, architecture internals relevant to fine-tuning, language coverage, failure modes, licensing. Each section: links → what the study covers → missed details → implications for THIS pipeline (glmocr SDK swap, Hungarian LoRA, vLLM/MIG serving, confidence stack). Contradictions with the study are flagged ⚠️. Compiled Aug 2026; items marked "verify at link" were not read line-by-line — check before relying.*

---

## 1. GLM-OCR (Zhipu / Z.ai)
**Links:** 📄 arXiv: https://arxiv.org/abs/2603.10910 · 💻 GitHub: https://github.com/zai-org/GLM-OCR · 🤗 https://huggingface.co/zai-org/GLM-OCR

**Study covers:** 0.9B CogViT+GLM-0.5B, MTP, two-stage PP-DocLayout-V3 pipeline, KIE, 8-language core, env-var config, license split.

**Missed technical details:**
- **MTP head mechanics:** trained to predict ~10 tokens/step; ~5.2 realized average at inference (~50% throughput gain). The same head is reused as the **speculative draft model** at serving — vLLM `--speculative-config '{"method":"mtp","num_speculative_tokens":3}'`, SGLang `NEXTN`. If you merge a LoRA, verify the MTP head survives the merge or speculative decoding silently degrades.
- **GRPO RL reward design** (directly reusable for your own fine-tune): rewards = Normalized Edit Distance (text), CDM (formula), TEDS (table), field-level F1 (KIE), **plus explicit penalties for repetition and malformed structure** — i.e., the anti-loop and format-conformity behavior is trained-in, not emergent. A format-native Hungarian fine-tune could copy this reward menu.
- **Connector:** lightweight cross-modal connector with efficient token downsampling + SwiGLU — region crops become very few tokens; this is why region-level requests are so cheap.
- **Prompt surface is a small fixed set** per region type (text / table→HTML / formula→LaTeX / KIE-with-schema); no free-form instruction following. The exact strings live in the SDK (`glmocr` package), not the paper — extract them from source for the adapter shim (verify at link).
- **SDK extras the study under-used:** `--layout-device cpu|cuda:N` CLI flag; `--set` dotted-path overrides; Flask server (`python -m glmocr.server`, list-of-images = pages of ONE document, one document per request); modular classes exported (`PageLoader`, `OCRClient`, `PPDocLayoutDetector`, `ResultFormatter`); agent "Skill" mode (`pip install glmocr` + API key).
- **Official LLaMA-Factory fine-tuning tutorial** exists in-repo (`examples/finetune/README.md`, Feb 2026).

**Implications:** the adapter shim should replicate the *fixed prompt set semantics*, not invent new ones; GRPO reward menu = template for your Hungarian RL/format-native run; MTP-head preservation is a fine-tune QA item.

---

## 2. PaddleOCR-VL v1 / v1.5 / v1.6 (Baidu)
**Links:** 📄 v1: https://arxiv.org/abs/2510.14528 · 📄 v1.5: https://arxiv.org/abs/2601.21957 · 📄 v1.6: https://arxiv.org/abs/2606.03264 · 📄 PaddleOCR 3.0 report: https://arxiv.org/abs/2507.05595 · 💻 https://github.com/PaddlePaddle/PaddleOCR · 🤗 https://huggingface.co/PaddlePaddle/PaddleOCR-VL-1.6 · Docs: https://www.paddleocr.ai/

**Study covers:** two-stage NaViT+ERNIE-0.3B, 109 languages, in-the-wild strength, CC≥8.0 floor, full-pipeline requirement.

**Missed technical details:**
- ✅ **Hungarian is EXPLICITLY listed** in the official supported-language list (Latin group: "…Croatian, Uzbek, **Hungarian**, Serbian (Latin)…" — docs FAQ, paddleocr.ai). The study's "probable, unverified" is now resolved: **verified**. Classic PP-OCRv5 Latin recognizer (32 languages) also lists Hungarian, with ONNX conversions in the wild.
- ⚠️ **CONTRADICTION — fine-tuning is NOT supported.** Official FAQ: "Currently, we do not support fine-tuning of the model, but it is a high-priority feature and will be released soon." The study rated Paddle FT as merely "less turnkey"; in fact there is no supported path today. If Hungarian needs a fine-tune, PaddleOCR-VL cannot be the target — this materially strengthens the Qwen3.5/DeepSeek fine-tune arms.
- **Chart recognition is OFF by default** — must set `use_chart_recognition=True`. A silent capability gap if you compare models without it.
- **Reading order:** PP-DocLayoutV2/V3 predicts reading order via a Global-Pointer-style pointer mechanism over detected regions, with multi-point/**polygonal** localization (handles skew/warp) — the layout stage outputs order, not just boxes; your assembly step should consume it rather than re-sorting by y-coordinate.
- **v1.6 = same architecture as v1.5** ("zero-cost migration"); gains come from **under-optimized region refinement + progressive post-training (RL)** — i.e., data/post-training only. Checkpoint swaps between 1.5↔1.6 are drop-in.
- **Table training corpus:** 5M+ image-table pairs via auto-annotation, annotation mining, and synthesis (v1 paper appendix) — explains the TableTEDS lead.
- **Serving invocation detail:** the pipeline drives the served VLM via `paddleocr … --vl_rec_backend vllm-server --vl_rec_server_url http://HOST:PORT/v1`; FastDeploy remains Baidu's fastest backend (~2.0 pages/s A100 vs ~1.38 vLLM).

**Implications:** Hungarian coverage confirmed → PaddleOCR-VL-1.6 is a legitimate no-fine-tune arm; but the no-fine-tune fact means it's the *ceiling-fixed* arm — if its Hungarian diacritics disappoint, you swap models rather than tune it. Turn chart recognition on for the bake-off.

---

## 3. DeepSeek-OCR & DeepSeek-OCR 2
**Links:** 📄 OCR: https://arxiv.org/abs/2510.18234 · 📄 OCR-2: https://arxiv.org/abs/2601.20552 · 📄 Independent critique: https://arxiv.org/abs/2601.03714 ("Visual Merit or Linguistic Crutch?") · 💻 https://github.com/deepseek-ai/DeepSeek-OCR · 🤗 https://huggingface.co/deepseek-ai/DeepSeek-OCR , /DeepSeek-OCR-2 · vLLM recipe: https://recipes.vllm.ai/deepseek-ai/DeepSeek-OCR · Unsloth: https://unsloth.ai/docs/models/tutorials/deepseek-ocr-how-to-run-and-fine-tune , /deepseek-ocr-2

**Study covers:** optical compression, MoE decoder, repetition failure mode, n-gram guardrail, Unsloth fine-tuning path.

**Missed technical details:**
- **Exact resolution modes & token budgets** (serving-critical): Tiny = 64 tokens @512², Small = 100 @640², Base = 256 @1024², Large = 400 @1280²; **Gundam / Gundam-Master** = tiled multi-local-view + one global view for dense pages. Your rasterization target should match the chosen mode's native size exactly.
- **Prompt grammar** (mode-selecting): `<image>\n<|grounding|>Convert the document to markdown.` (layout-aware) · `<image>\nFree OCR.` (plain text) · `<image>\nParse the figure.` · `<image>\nLocate <|ref|>xxx<|/ref|> in the image.` The Ollama card warns the model is **input-sensitive: a missing punctuation mark or newline in the prompt can corrupt output** — pin prompt strings byte-exactly in the gateway.
- **Compression ablation:** 97% precision at <10× text-token:vision-token ratio, ~60% at 20× (Fox benchmark) — i.e., accuracy degrades predictably with page text density per mode; dense pages need Base/Large/Gundam, not Small.
- **OCR-2 internals:** DeepEncoder V2 is a **Qwen2-0.5B-based** "Visual Causal Flow" module that *reorders visual tokens into learned human reading order* before the decoder; reading-order edit distance 0.085→0.057; decoder unchanged (3B-MoE, ~500M active), 256–1120 visual tokens/page. Repetition, self-measured in production: 6.25%→4.17% (user-log images), 3.69%→2.88% (PDF pipeline).
- **Critique paper (2601.03714) finding:** the model leans on language priors — it "corrects" visually-clear but semantically-odd words. For your validators this means: **numeric/ID fields extracted by DeepSeek-lineage models need the value-in-OCR/checksum cross-check most**, because fluent silent correction is trained-in behavior.
- **Fine-tuning gotcha:** the stock checkpoint **does not run/train on current transformers**; Unsloth ships a modified upload (`unsloth/DeepSeek-OCR[-2]`, Stranger Vision patches) — fine-tune from *that*, not `deepseek-ai/…`.
- **Training data breadth:** charts, chemical structures (SMILES output!), geometry — it can emit SMILES for chemistry regions, a capability the study never mentioned.
- **Licensing:** v1 code MIT; OCR-2 repo Apache-2.0; the toolchain imports **AGPL PyMuPDF** (repo issue #223) — replicate rasterization with pypdfium2 instead of vendoring their scripts.

**Implications:** mode/DPI matching goes into the gateway config per model; the byte-exact prompt constant goes into CI; Unsloth's modified checkpoint is the fine-tune base; SMILES/figure-parse modes are a bonus route for scientific docs.

---

## 4. Baidu Unlimited-OCR
**Links:** 📄 arXiv: https://arxiv.org/abs/2606.23050 ("Unlimited OCR Works") · 💻 https://github.com/baidu/Unlimited-OCR · 🤗 https://huggingface.co/baidu/Unlimited-OCR · vLLM recipe: https://recipes.vllm.ai/baidu/Unlimited-OCR

**Study covers:** R-SWA concept, one-shot multi-page, continue-training lineage, serving flags, 32K budget.

**Missed technical details:**
- **R-SWA precise mechanics:** every generated token attends to (a) **all reference tokens** — visual tokens + prompt — held *statically outside state transitions*, and (b) only the **preceding n output tokens, n = 128 by default**. The KV cache is implemented as a **queue of capacity m + n** (m = reference length): each new token evicts the oldest *output* token's KV. Cache ceiling = L_m + n vs O(T) for vanilla attention.
- **Why not vanilla SWA/linear attention:** in those, visual tokens get pulled into recursive state updates and image features **progressively blur** as pages advance; R-SWA's exclusion of reference tokens from the sliding state is the whole trick. (Also why the fine-tune recipe freezes the encoder.)
- **Long-horizon numbers:** 40+ pages in one call, edit distance <0.11, **Distinct-20 = 96.08% / Distinct-35 = 96.90%** — the paper's own anti-loop metric. Beyond ~40 pages you hit the 32K budget; **max_length=32768 covers prompt + output together** (so multi-page vision tokens eat into the output budget — the study's ~800-tokens/page heuristic should subtract reference tokens too).
- **Continue-train recipe verbatim** (your Hungarian template): 4,000 steps from the DeepSeek-OCR checkpoint, DeepEncoder frozen, decoder-only, ~2M document samples, **9:1 single-page:multi-page split with multi-page samples built by simple concatenation**, on 8×16 A800.
- **Repetition controls:** `no_repeat_ngram_size=35`, `ngram_window=128` (use `1024` for multi-page/PDF); prompt must begin literally `<image>` (`<image>document parsing.` / `<image>Multi page parsing.`); raw output carries `<|ref|>…<|/ref|>` text and `<|det|>…<|/det|>` coordinate boxes — unwrap ref, drop det for clean markdown.
- **Roadmap stated in paper:** 128K-context training run and a "prefill pool" that fetches document chunks on demand (human page-flipping analogue); R-SWA positioned as general-purpose for ASR/translation — i.e., expect a longer-context v2; don't over-engineer chunking now.

**Implications:** your 25–30-page chunking rule gets a correction (budget = 32K − reference tokens − prompt); the 9:1 concatenation recipe is directly copyable if you ever long-horizon-tune a Hungarian variant; the eviction-queue design explains why per-request `window_size` matters and must match the serving engine's processor args.

---

## 5. dots.ocr (rednote-hilab / Xiaohongshu)
**Links:** 💻 https://github.com/rednote-hilab/dots.ocr (tech notes: `assets/blog.md`) · 🤗 https://huggingface.co/dots-studio/dots.ocr (2026 lineage: dots.mocr / v1.5)

**Study covers:** 1.7B single VLM, JSON+bboxes, blind-vote strength, repetition on `…`/`___`, slow throughput.

**Missed technical details:**
- **Prompt-switchable task modes** in one model: full layout+recognition JSON, layout-only detection, recognition-only, grounding — mode selection is by prompt, so your adapter can request layout-only from dots as a *second-opinion layout signal* for the confidence stack.
- **Repo-documented limitations** (verbatim class): endless repetition triggered by runs of continuous special characters (ellipses, underscores — think dotted TOC leader lines and form blanks: exactly Hungarian invoice/form furniture); **in-document pictures are not parsed** at all; **local model directory name must contain no periods** (`DotsOCR`); transformers path markedly slower than vLLM (`use_hf=True`).
- **2026 lineage:** renamed dots.mocr / v1.5 adds **SVG parsing of charts/graphics** — a structured-chart route no other candidate offers.
- vLLM officially integrated since v0.11.0 (standard `vllm/vllm-openai` image works).

**Implications:** pre-strip or regex-guard TOC leader-dots and underscore blanks before sending pages to dots.ocr; its layout-only mode is a free cross-check input for EXTRACTCONF-style spatial-divergence features.

---

## 6. MinerU 2.5 / Pro / Diffusion (OpenDataLab)
**Links:** 📄 arXiv: https://arxiv.org/abs/2509.22186 · 💻 https://github.com/opendatalab/MinerU · 🤗 https://huggingface.co/opendatalab/MinerU2.5-2509-1.2B

**Study covers:** platform backends, 1.2B two-stage VLM, router, Latin/CJK bias, license change.

**Missed technical details:**
- **The two stages live inside ONE 1.2B VLM** (not a separate detector): stage 1 = layout analysis on a *downsampled* global image; stage 2 = recognition on **native-resolution crops** guided by that layout — the coarse-to-fine trick that dodges high-res compute. Different from Paddle/GLM's separate-detector design; no layout-tier placement decision exists for MinerU.
- **Full content integrity by design:** headers, footers, page numbers are *preserved and labeled* (standardized schema), not dropped — most competitors discard them; matters for legal/compliance documents where page furniture is evidentiary.
- **MinerU2.5-Pro (95.69 v1.6) is data-engineering only** — identical 1.2B architecture; the +5 TableTEDS came from the data engine. Independent evidence that your 10–30K-pair data plan can move numbers without architecture work.
- **Dynamic repetition suppression:** per-layout-element frequency/presence penalties applied at decode — a third repetition-control pattern (vs n-gram processors and R-SWA) worth borrowing conceptually.
- **MinerU-Diffusion:** decodes blocks of 32 masked tokens, iteratively unmasking by **per-position confidence** — native confidence signal, but requires the custom `nano_dvlm` engine (block-wise diffusion denoising), incompatible with vLLM. Quarterly watch-item for the confidence stack.
- Ecosystem: MCP server, LangChain/Dify/FastGPT integrations, MLX (100–200% speedup on Apple Silicon), 10+ Chinese domestic accelerators.

**Implications:** if MinerU enters your fleet it needs no layout tier; its header/footer preservation makes it the archival/compliance arm; the Pro result is your strongest argument that fine-tuning data > architecture for this class.

---

## 7. HunyuanOCR-1.5 (Tencent)
**Links:** 📄 arXiv: https://arxiv.org/abs/2607.04884 · 💻 Tencent-Hunyuan GitHub org (repo name varies — verify at link) · 🤗 Hunyuan HF org

**Study covers:** ~1B, 94.74 v1.6, top-open MORE multilingual, faithfulness.

**Missed technical details:**
- **CHAOS-Bench 14.15 vs peers <7** quantifies *faithfulness*: it reproduces the source (including errors/odd spellings) rather than silently correcting — the exact opposite failure profile of the DeepSeek "linguistic crutch". For high-stakes fields, a faithful model + text-LLM correction beats a fluent model that already corrupted the evidence.
- Coordinate/grounding outputs + **text-image translation** as a built-in task (source-language page → target-language text) — a route for Hungarian→English delivery without a second hop.
- Repo notes a **vLLM-vs-transformers accuracy gap under repair** — pin and A/B the engines before trusting vLLM numbers for it.

**Implications:** strongest candidate for the "evidence-preserving" arm of the confidence architecture; verify engine parity before the bake-off scores count.

---

## 8. olmOCR-2 (Allen AI)
**Links:** 💻 https://github.com/allenai/olmocr · 🤗 https://huggingface.co/allenai (olmOCR-2 + olmOCR-bench) · 📄 olmOCR technical reports linked from repo (verify current arXiv ID at link)

**Missed technical details:**
- **RLVR training with unit-test-style rewards:** binary verifiable checks on outputs (structure parses, content present) as RL signal — a third reward paradigm (vs GRPO metric-rewards and SFT) directly relevant to your format-native fine-tune: you already plan CI fixtures; RLVR turns those fixtures into training rewards.
- **Pipeline heuristics** around the model: automatic page retries, quality filters, perplexity-style rejection — the productionized version of your hedging/loop-detector layer, worth reading as prior art.
- English-only by design; 7B, H100-class; olmOCR-bench is *theirs* — treat its leaderboard placement of competitors accordingly.

**Implications:** not a Hungarian candidate, but two of its ideas (RLVR-from-fixtures, retry heuristics) belong in your pipeline regardless.

---

## 9. LightOnOCR-2-1B (LightOn)
**Links:** 🤗 https://huggingface.co/lighton (model card + blog — verify exact repo at link)

**Missed technical details:** European-language training focus (the only candidate *marketed* on it), ~42.8 pages/s on H100 claim, and a **bbox variant** emitting bounding boxes for embedded images. Whether Hungarian is in its training distribution is **unverified** — the model card / blog is the check. Standard autoregressive, vanilla-vLLM servable.

**Implications:** the cheapest possible Hungarian probe (1B, vanilla serving); if its card confirms Hungarian, it jumps the queue for the bake-off.

---

## 10. Qwen3-VL & Qwen3.5 (Alibaba)
**Links:** 📄 Qwen3-VL: https://arxiv.org/abs/2511.21631 · 💻 https://github.com/QwenLM/Qwen3-VL · 💻 family repo: https://github.com/QwenLM/Qwen3.8 · Qwen3.5 blog: https://qwen.ai/blog?id=qwen3.5 · 🤗 https://huggingface.co/Qwen/Qwen3.5-4B · Unsloth: https://unsloth.ai/docs/models/qwen3.6

**Study covers:** 32/39-language OCR, Qwen3.5 native multimodal + 201 languages, size ladder, MTP.

**Missed technical details:**
- **Qwen3-VL language facts, precisely:** 30M in-house multilingual OCR samples across **39 languages**; >70% accuracy on **32 of 39** (their usability bar); **the actual language list is Figure 2 of arXiv 2511.21631 — Hungarian membership must be read off that figure** (verify at link). Note the marketing inconsistency: HF cards say "32 (up from 19)", GitHub says "(up from 10)" — the paper's framing (10 non-EN/ZH → 39 tested) is authoritative.
- **Image token budgeting (serving-critical):** ~32× spatial compression; per-image visual tokens steerable via processor pixel budget (e.g., 256–1280 tokens: `processor.image_processor.size = {"longest_edge": 1280*32*32, "shortest_edge": 256*32*32}`); explicit dimensions round to **multiples of 32** (28 for Qwen2.5-VL). For region crops, cap the budget low — this is your cost knob in the glmocr-swap.
- **Architecture bits relevant to fine-tuning:** SigLIP-2 vision encoder + 2-layer MLP merger compressing 2×2 features → 1 token + **DeepStack** multi-level feature injection + Interleaved-MRoPE; grounding uses a normalized **[0,1000] coordinate system**. Training Stage-0 updates *only the MLP merger* (encoder + LLM frozen) — corroborates the freeze-the-encoder LoRA plan; the merger is a legitimate additional LoRA target.
- **Qwen3.5 serving implications:** hybrid **Gated DeltaNet + Gated Attention** in an 8×(3×DeltaNet→FFN→1×Attention→FFN) pattern — linear-attention layers change KV-cache behavior; pin a vLLM version with confirmed Qwen3.5 support and re-test MIG-slice memory profiles (the study's KV heuristics assume standard attention). The "201 languages" claim covers text/dialect competence — **OCR-specific Hungarian accuracy remains an empirical question for the probe set**, exactly like every other candidate.
- Qwen document parsing emits "QwenVL HTML" with bbox metadata — the dialect your adapter normalizes away.

**Implications:** the pixel-budget knob replaces `max-model-len` as the primary cost control in the SDK-swap; Fig-2 check is a 5-minute task that could re-rank Qwen3-VL; MIG memory re-validation is mandatory for Qwen3.5's hybrid attention.

---

## 11. Adjacent components (implementation-relevant only)
- **PP-DocLayout-V3** — 🤗 https://huggingface.co/PaddlePaddle/PP-DocLayoutV3 (Apache-2.0): RT-DETR-class detector with polygonal localization + reading-order pointer; **the ONNX/OpenVINO export path for V3 specifically remains the unverified prerequisite for the CPU layout tier** — confirm before committing Tier 1.
- **Qianfan-OCR** — 📄 https://arxiv.org/abs/2603.13398: its Table 6 is the definitive end-to-end vs two-stage comparison (two-stage OCR+Qwen3-4B: CharXiv 0.0 vs 94.0 end-to-end) — the citation behind the study's "visual questions die at the OCR→text boundary."
- **OmniDocBench** — 📄 https://arxiv.org/abs/2412.07626: the benchmark's own paper; read §data-composition to understand the Latin/CJK-print bias the study asserts.
- **ConfBench** — 📄 https://arxiv.org/abs/2608.01792: verbalized vs logprob confidence for KIE, three input-modality conditions; their appendix A.5 workflow is a ready-made template for your confidence evaluation harness.
- **EXTRACTCONF** — 📄 https://arxiv.org/abs/2606.24420: the 40-feature menu (logprob mean/min/P10/std, entropy mean/max/P90 per call; value-in-OCR match; neighborhood-text overlap; spatial centroid divergence; OCR confidence; image quality) fused by CatBoost with optional post-hoc recalibration — your §3-feature shortcut (value-in-OCR + logprob-min + validator) comes from here; the full feature list is the upgrade path.
- **DeepSeek-OCR critique** — 📄 https://arxiv.org/abs/2601.03714: the "linguistic crutch" evidence base.

---

## Consolidated link index

**arXiv**
| Paper | Link |
|---|---|
| GLM-OCR Technical Report | https://arxiv.org/abs/2603.10910 |
| PaddleOCR-VL (v1) | https://arxiv.org/abs/2510.14528 |
| PaddleOCR-VL-1.5 | https://arxiv.org/abs/2601.21957 |
| PaddleOCR-VL-1.6 | https://arxiv.org/abs/2606.03264 |
| PaddleOCR 3.0 Technical Report | https://arxiv.org/abs/2507.05595 |
| DeepSeek-OCR (Contexts Optical Compression) | https://arxiv.org/abs/2510.18234 |
| DeepSeek-OCR 2 (Visual Causal Flow) | https://arxiv.org/abs/2601.20552 |
| Visual Merit or Linguistic Crutch? (critique) | https://arxiv.org/abs/2601.03714 |
| Unlimited OCR Works (R-SWA) | https://arxiv.org/abs/2606.23050 |
| MinerU2.5 | https://arxiv.org/abs/2509.22186 |
| HunyuanOCR-1.5 | https://arxiv.org/abs/2607.04884 |
| Qwen3-VL Technical Report | https://arxiv.org/abs/2511.21631 |
| Qianfan-OCR | https://arxiv.org/abs/2603.13398 |
| OmniDocBench | https://arxiv.org/abs/2412.07626 |
| ConfBench | https://arxiv.org/abs/2608.01792 |
| EXTRACTCONF | https://arxiv.org/abs/2606.24420 |

**GitHub / model hubs**
| Repo | Link |
|---|---|
| GLM-OCR | https://github.com/zai-org/GLM-OCR |
| PaddleOCR (incl. VL, PP-DocLayoutV3, docs) | https://github.com/PaddlePaddle/PaddleOCR · https://www.paddleocr.ai/ |
| DeepSeek-OCR / OCR-2 | https://github.com/deepseek-ai/DeepSeek-OCR |
| Unlimited-OCR | https://github.com/baidu/Unlimited-OCR |
| dots.ocr | https://github.com/rednote-hilab/dots.ocr |
| MinerU | https://github.com/opendatalab/MinerU |
| olmOCR | https://github.com/allenai/olmocr |
| Qwen3-VL / family | https://github.com/QwenLM/Qwen3-VL · https://github.com/QwenLM/Qwen3.8 |
| Qwen3.5 blog | https://qwen.ai/blog?id=qwen3.5 |
| vLLM recipes (DeepSeek, Unlimited, Paddle) | https://recipes.vllm.ai/ |
| Unsloth guides (DeepSeek-OCR/2, Qwen3.x) | https://unsloth.ai/docs/ |
| HF: PP-DocLayoutV3 | https://huggingface.co/PaddlePaddle/PP-DocLayoutV3 |
| HF: baidu/Unlimited-OCR · zai-org/GLM-OCR · PaddlePaddle/PaddleOCR-VL-1.6 · deepseek-ai/DeepSeek-OCR(-2) · opendatalab/MinerU2.5-2509-1.2B · Qwen/Qwen3.5-4B · unsloth/DeepSeek-OCR | https://huggingface.co/ + path |

---

## Top 10 missed details, ranked by impact on this pipeline
1. ✅ **Hungarian is verified in PaddleOCR-VL's language list** — upgrades it from "probable" to a confirmed no-fine-tune arm.
2. ⚠️ **PaddleOCR-VL fine-tuning is officially unsupported** — contradicts the study; removes it as a fine-tune target and strengthens Qwen3.5/DeepSeek arms.
3. **Unsloth fine-tuning requires the modified `unsloth/DeepSeek-OCR` checkpoint** — training from the stock repo fails on current transformers.
4. **Unlimited-OCR's 32K budget covers prompt + reference + output together** — the chunking heuristic must subtract vision/prompt tokens; and n=128 window / queue-eviction mechanics must match per-request `window_size`.
5. **Qwen3-VL's visual-token pixel budget (256–1280 tokens, 32× compression, ×32 dimension rounding)** — the real cost knob for region crops in the SDK swap; and the Hungarian answer sits in Figure 2 of the paper, a 5-minute check.
6. **GLM-OCR's GRPO reward menu (NED/CDM/TEDS/F1 + repetition & malformed-structure penalties)** — a copyable objective design for the format-native Hungarian fine-tune; MTP-head survival is a LoRA-merge QA item.
7. **DeepSeek prompt grammar is byte-sensitive** (documented) and mode/token budgets are fixed per resolution — prompt constants and mode-matched raster sizes belong in CI.
8. **Qwen3.5's Gated-DeltaNet hybrid attention invalidates standard KV-cache assumptions** — re-validate MIG memory profiles and pin vLLM versions before fleet math.
9. **MinerU2.5-Pro's +5-point gain is data-engineering only** (identical architecture) — the strongest published evidence that your data plan can outperform architecture chasing.
10. **HunyuanOCR's faithfulness (CHAOS 14.15) vs DeepSeek's "linguistic crutch"** are opposite failure poles — pick per field-criticality, and aim the value-in-OCR validator hardest at the fluent-corrector lineage.

## Caveats
This review was compiled from the papers' abstracts/HTML excerpts, official docs, model cards, and repo READMEs/issues gathered across this project — not cover-to-cover reads. Items tagged "verify at link" (GLM prompt strings in source, Qwen Fig-2 language list, LightOn training distribution, HunyuanOCR repo path, olmOCR-2 arXiv ID, PP-DocLayoutV3 export formats) are precisely the ones to check by hand. Repos and model cards on this list have changed monthly all year; treat every flag/parameter as "as of Aug 2026, re-verify on your pinned versions."
