---
aliases: ["ColPali"]
tags: [rag, vision-retrieval, late-interaction, licensing, model]
sources: [The Everything Else Map of SOTA RAG Optimization (2024–2026) — A Decision-Ready Survey for a Greenfield Agentic RAG System.md, Deep-Dive — 10 RAG Optimization Techniques for a Self-Hosted Agentic Stack.md]
created: 2026-08-27
updated: 2026-08-27
---

# ColPali

ColPali (arXiv 2407.01449, ICLR 2025) applies ColBERT-style **late interaction (MaxSim)** over patch-level embeddings from a VLM, indexing document **page images directly — no OCR, no layout pipeline**. Query text tokens are matched against page patch vectors.

## Why it is the biggest untapped opportunity for a VDU pipeline

If you already render page images and run layout detection, adding ColPali retrieval is a **low-friction extension that bypasses OCR brittleness entirely** for tables, charts and scanned pages. [[Qdrant]] natively supports the multi-vector representation it needs.

The decision threshold is explicit: **if OCR quality on your VDU pipeline already yields >~95% correct answers on table/chart questions, defer ColPali.** Its value is proportional to your OCR's failure rate on visually rich pages.

## ColPali is an architecture, not a model

**This is the single most important thing about it, and it is easy to miss.** ColPali is late interaction *applied to* a vision-language backbone. The backbone is a config value — exactly the same inversion as [[Pipeline as Platform, Model as Config]] on the OCR side, and the same reason the [[glmocr SDK]] outlived its model. "Should we use ColPali?" and "which backbone?" are two separate decisions, and only the second one churns as models churn.

The consequence here is concrete: **the current top-scoring checkpoint runs on Qwen3.5-4B** — the same backbone family as the recognition arm on the [[Hungarian Model Decision Matrix]]. One model family could serve both the recognition slot and the retrieval slot, sharing serving patterns, VRAM profiles and the Gated-DeltaNet caveat in [[Qwen Model Family]].

## The family, verified

Read from the upstream `illuin-tech/colpali` README on **2026-08-27**. ViDoRe v1 scores; the models are also evaluated on ViDoRe v2 (out-of-domain, arXiv 2505.17166) and a newer v3 mix.

| Checkpoint | Version | Backbone | Licence | ViDoRe |
|---|---|---|---|---|
| `vidore/colpali` | v1.0 → **v1.3** | PaliGemma-3B-mix-448 | ❌ **Gemma** — not OSI-permissive | 81.3 → **84.8** |
| `vidore/colqwen2` | v0.1 → **v1.0** | Qwen2-VL-2B-Instruct | Apache-2.0 ✅ | 87.3 → **89.3** |
| `vidore/colqwen2.5` | v0.1 → **v0.2** | Qwen2.5-VL-3B-Instruct | Apache-2.0 ✅ (but see below) | 88.8 → **89.4** |
| `vidore/ColSmol` | 256M / 500M | SmolVLM | Apache-2.0 ✅ | 80.1 / 82.3 |
| `vidore/ModernVBERT` | v1.0 | ModernVBERT (~250M) | — | within 0.6 NDCG@5 of ColPali at ~10× fewer params |
| `TomoroAI/tomoro-colqwen3-embed-4b` | 4B (an 8B is also published) | Qwen3-VL | Apache-2.0 ✅ | **90.6** |
| `athrael-soju/colqwen3.5-4.5B-v3` | v3 | **Qwen3.5-4B** | Apache-2.0 ✅ | **90.9** — top of the table |

**The two best checkpoints are community-published, not vidore-published.** That is a supply-chain fact, not a quality judgement: `vidore/` tops out at ColQwen2.5-v0.2 (89.4), and the +1.5 points above it come from third-party repos. Weigh it the way you would weigh any unmaintained dependency, and pin the revision hash.

Also on the shelf: ColNomic, jina-embeddings-v4 multivector, and the multilingual community forks (`tsystems/colqwen2.5-3b-multilingual`, `Metric-AI/ColQwen2.5-7b-multilingual`).

## Licensing — the main risk, verify per checkpoint

Because the backbone is a config value, **the licence is a property of the checkpoint, not of "ColPali"**. Three distinct situations:

- **The original is not permissive.** ColPali's LoRA adapters are MIT, but the PaliGemma backbone is **Gemma-licensed**. Under a permissive-only rule the whole `vidore/colpali` column is out — which is why the ColQwen line matters rather than being a mere upgrade. — **ColQwen2.5 carries a documented licence conflict.** The upstream README lists `vidore/colqwen2.5-v0.2` as **Apache-2.0**; the model card states the Qwen2.5-VL backbone is under the **Qwen RESEARCH LICENSE AGREEMENT** with MIT adapters. **Both were read on 2026-08-27 and they disagree.** Base `Qwen2.5-VL-3B/7B-Instruct` are genuinely Apache-2.0 (the 72B is not), so the README is probably right — but "probably right" is not a licence position. Resolve it against the exact revision you deploy, in writing. — **Community checkpoints self-report Apache-2.0** (`yydxlv/colqwen2.5-7b`, `tsystems/colqwen2.5-3b-multilingual`, `Metric-AI/ColQwen2.5-7b-multilingual`, and both Qwen3/Qwen3.5 entries above) with MIT adapters.

The training set `vidore/colpali_train_set` is **CC-BY-NC-4.0** — relevant only if you retrain, and a real constraint if you plan to fine-tune on Hungarian pages. See [[Permissive Licensing Constraints]].

## The storage problem and its fix

ColPali emits **~1,030 vectors per page** (ColQwenX doubles it) — at 128 dims × float32 that is ~527 KB/page uncompressed. HNSW build cost grows quadratically with vectors per page, so pooling is mandatory at scale.

**Token pooling + two-stage retrieval** is the answer. Qdrant's measured result: mean-pool by grid rows, prefetch top-200 on pooled vectors, exact-MaxSim rerank on full multivectors → **13× faster retrieval** while preserving **NDCG@20 = 0.952, Recall@20 = 0.917**. Max pooling underperformed. The Visual RAG Toolkit paper (arXiv 2602.12510) reduces ~1024 → ~32 vectors/page (a fixed 32× via row/column pooling) with ~4× throughput and negligible quality loss at k≤10, and adds **"token hygiene"** — strip BOS/EOS/prompt/padding tokens, which distort MaxSim.

Binary quantization reduces memory but **not** the number of MaxSim comparisons; only pooling does that.

## Serving

**`colpali-engine` is deprecated.** Verbatim from the upstream README, read 2026-08-27: *"`colpali-engine` is now deprecated. The repository and package remain available for research, reproducibility, and existing projects, but we recommend Sentence Transformers for new projects and production use."* Any design naming `colpali-engine` as the inference path is already out of date — build against **Sentence Transformers** instead. The models are unaffected; only the serving library moved.

Generation over the retrieved pages runs on a VLM on [[vLLM]] as before. For clean digital text, a strong text pipeline ([[Qwen3-Embedding]] + BM25) is cheaper and usually sufficient.

## Hungarian — the honest position

**No ColPali variant declares Hungarian.** The multilingual forks train on `llamaindex/vdr-multilingual-train` plus German/English VQA sets; `tsystems/colqwen2.5-3b-multilingual` covers **5 languages and Hungarian is not among them**.

But the failure mode is different from OCR, and that difference is the point:

- Visual retrieval never has to *recognise* ő and ű. It has to match a **Hungarian query** against page patches. The risk moves from the vision side to the **query-encoder text side** — where the Qwen3.5 backbone claims 201 languages. — So the realistic risk is **cross-lingual drift**, not diacritic failure: a query encoder that handles Hungarian text in general but was never trained to align Hungarian queries with document patches. — It is measurable on the same artifact as everything else: Hungarian questions against your own pages. See [[Golden Set and Eval Harness]] and [[RAG Evaluation]].

This is a structurally better position than the OCR side is in — the double-acute problem does not exist here — but it is untested, not proven.

## Related

[[Qdrant]] · [[Two-Stage Retrieve-Then-Rerank]] · [[The OCR-to-Text Boundary Limit]] · [[Permissive Licensing Constraints]] · [[Pipeline as Platform, Model as Config]] · [[Qwen Model Family]] · [[Hungarian Model Decision Matrix]]
