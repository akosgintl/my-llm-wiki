# Deep-Dive: 10 RAG Optimization Techniques for a Self-Hosted Agentic Stack

**Scope:** Implementation-ready analysis for a self-hosted agentic RAG system on AKS with vLLM, Qdrant (dense/multivector), OpenSearch (BM25/sparse), Neo4j (document graph), LangGraph orchestration, bge-reranker-v2-m3, and a Hungarian + English corpus under permissive-license constraints (Apache/MIT/BSD).

## TL;DR
- **Eight of the ten techniques are directly production-ready on your stack today with permissive licenses.** The two license risks are **Provence** (`naver/provence-reranker-debertav3-v1` is CC-BY-NC-4.0, non-commercial — do NOT deploy the weights commercially) and the **ColPali/ColQwen family** (LoRA adapters are MIT but the VLM backbone license varies: PaliGemma is Gemma-licensed, and ColQwen2.5's Qwen2.5-VL backbone shows conflicting Apache-2.0 vs "Qwen Research License" tags across checkpoints — verify per checkpoint).
- **Highest ROI, lowest risk to implement first:** Anthropic Contextual Retrieval (reimplemented with your own vLLM small model), Qwen3-Embedding (Apache-2.0, 100+ languages incl. Hungarian, native vLLM embed support), and MRL + int8 quantization + rescoring in Qdrant — these compound and require no non-permissive components.
- **The most stack-specific wins** are LMCache/CacheBlend KV-cache reuse (turns RAG prefill from full recompute into near-100% cache hits on vLLM) and hybrid learned-sparse — but note learned-sparse Hungarian coverage is thin, so BGE-M3 sparse weights or OpenSearch multilingual-v1 are your realistic Hungarian options rather than English-only SPLADE.

## Key Findings
- Contextual Retrieval's reported gains (35% / 49% / 67% failure-rate reductions) are from Anthropic's own internal eval using 1−recall@20 with Gemini Text 004 embeddings + Cohere reranker, not your exact stack — treat as directional and re-run evals locally.
- Qwen3-Embedding-8B ranked No.1 on MTEB Multilingual as of June 5, 2025 (score 70.58), is Apache-2.0, supports 32K context and Matryoshka truncation down to 32 dims, uses last-token pooling, and requires an instruction prefix on queries only.
- CacheBlend (EuroSys 2025 Best Paper) and its production home LMCache reached production maturity (LMCache "graduated to production in January 2026"); LMCache integrates with vLLM v1 via the KV-connector API and offers CPU/disk/remote tiers.
- LettuceDetect (MIT) is authored by a Budapest/Vienna team, uses ModernBERT token classification, hits 79.22% example-level F1 on RAGTruth, runs 30–60 samples/s/GPU, and its v2 line adds 14-language PsiloQA coverage including Hungarian — an unusually good fit for your bilingual corpus.
- RAPTOR gives up to +20% absolute accuracy on QuALITY for holistic/multi-step questions; collapsed-tree retrieval beats tree-traversal; the tree does not update incrementally, which is the key operational caveat when mapping it onto your Neo4j graph.

## Details

---
### 1. Anthropic Contextual Retrieval (contextual embeddings + contextual BM25)

**How it works (implementation-ready).** For each chunk, prepend a 50–100 token LLM-generated context snippet that situates the chunk in its parent document, *before* embedding and *before* building the BM25 index. The exact prompt Anthropic published (Claude 3 Haiku) is:

```
<document>
{{WHOLE_DOCUMENT}}
</document>
Here is the chunk we want to situate within the whole document
<chunk>
{{CHUNK_CONTENT}}
</chunk>
Please give a short succinct context to situate this chunk within the overall document for the purposes of improving search retrieval of the chunk. Answer only with the succinct context and nothing else.
```

Full pipeline: chunk (a few hundred tokens) → contextualize → dual-index (dense embeddings + BM25) → retrieve top-150 via rank fusion (RRF) → rerank → top-20 into the LLM prompt.

**Reported numbers.** Per Anthropic (anthropic.com/engineering/contextual-retrieval), metric = 1−recall@20 with Gemini Text 004, retrieving top-20: "Contextual Embeddings reduced the top-20-chunk retrieval failure rate by 35% (5.7% → 3.7%). Combining Contextual Embeddings and Contextual BM25 reduced [it] by 49% (5.7% → 2.9%)"; adding reranking (Cohere) reduced the failure rate by 67% (5.7% → 1.9%).

**Prompt-caching economics.** With prompt caching (load the document once, reference the cached content per chunk), "the one-time cost to generate contextualized chunks is $1.02 per million document tokens" (assuming 800-token chunks, 8k-token docs, 50-token instruction, 100-token context per chunk).

**Self-hosted replication (your stack).** Replace Claude+prompt-caching with a small instruct model on vLLM (e.g., Qwen3-4B/8B-Instruct or Llama-3.1-8B). vLLM's automatic prefix caching gives you the equivalent of prompt caching: put the whole document as a shared prefix so per-chunk generation reuses the document KV cache. Generate context, prepend, then dual-write to Qdrant (dense) + OpenSearch (contextual BM25). For Hungarian, write the contextualizer instruction in the document's language or bilingually and validate output quality.

**Open-source reimplementations.** Anthropic cookbook (capabilities-contextual-embeddings-guide); Together AI's line-by-line open-model implementation (docs.together.ai); LlamaIndex and LangChain contextual-retrieval templates; RAGFlow context augmentation.

**Limitations / critiques.** Anthropic itself found generic document-summary prepending (an older technique) gave "very limited gains" and summary-based indexing "low performance" — the win comes specifically from chunk-specific context. Costs scale with corpus churn (every re-chunk re-contextualizes). Practitioners report the LLM sometimes emits boilerplate ("This chunk discusses...") that adds noise; constrain output and eval.

**Getting started.** (1) Pick a vLLM small model; (2) implement the prompt above with the whole doc as a cached prefix; (3) prepend context, embed with Qwen3-Embedding, BM25-index in OpenSearch; (4) RRF fuse + bge-reranker-v2-m3; (5) A/B against non-contextual baseline on a local eval set using 1−recall@20.

**Primary sources:** anthropic.com/engineering/contextual-retrieval · platform.claude.com/cookbook/capabilities-contextual-embeddings-guide · docs.together.ai/docs/how-to-implement-contextual-rag-from-anthropic

---
### 2. Qwen3-Embedding (0.6B / 4B / 8B)

**Architecture & training.** Built on Qwen3 dense foundation models. Multi-stage pipeline: large-scale weakly-supervised contrastive pre-training on LLM-synthesized data, supervised fine-tuning on high-quality labeled data, then model merging (slerp-style checkpoint merging) for robustness. Uses **last-token pooling** (tokenizer `padding_side='left'`; embeddings L2-normalized). Instruction-aware and MRL-capable.

**Scores (MTEB Multilingual / MMTEB "Mean (Task)").** Per Qwen's official blog (qwenlm.github.io/blog/qwen3-embedding): "The 8B size embedding model ranks No.1 in the MTEB multilingual leaderboard (as of June 5, 2025, score 70.58)." Full series: 0.6B = 64.33; 4B = 69.45; 8B = **70.58**. MTEB-English v2 Mean (Task): 0.6B = 70.70; 4B = 74.60; 8B = 75.22. C-MTEB (Chinese) Mean (Task): 0.6B = 66.33; 4B = 72.27; 8B = 73.84. (Leaderboard rank is a point-in-time claim; the GitHub README dates comparison scores to June 6, 2025 while the 8B card dates them May 24, 2025 — treat rank as of mid-2025.)

**Specs per size.** All three: 32K context, MRL support (user-defined output dims from **32** up to native). Native dims: 0.6B = 1024, 4B = 2560, 8B = 4096. Layers: 0.6B = 28; 4B/8B = 36.

**Instruction format (queries only).**
```python
def get_detailed_instruct(task, query):
    return f'Instruct: {task}\nQuery:{query}'
```
Documents get NO instruction ("# No need to add instruction for retrieval documents"). Instruction use yields ~1–5% improvement; the docs recommend writing instructions in English even for multilingual corpora (most training instructions were English).

**Multilingual / Hungarian.** 100+ languages. Hungarian is explicitly listed (Uralic family: Finnish, Estonian, Hungarian) in the GitHub README language table. No per-language Hungarian score is published in headline materials — validate on a local Hungarian eval set.

**Serving on vLLM.** `vllm serve Qwen/Qwen3-Embedding-8B --runner pooling` (newer vLLM) or `--task embed` (vLLM ≈0.8.5). Offline: `LLM(model=..., task="embed")`. Requires `vllm>=0.8.5`, `transformers>=4.51.0` (else `KeyError: 'qwen3'`). Multi-GPU: add `distributed_executor_backend="mp"`. vLLM defaults to last-token pooling + normalization, matching the model.

**License.** Apache-2.0 for all three sizes (confirmed in arXiv abstract and all HF cards).

**Comparison vs BGE-M3 / multilingual-e5.** Qwen3-Embedding leads both on MMTEB aggregate. BGE-M3 remains attractive because it natively produces dense + sparse + ColBERT-multivector in one pass and has strong Hungarian coverage; multilingual-e5-large is smaller/cheaper. For a bilingual corpus with permissive-license needs, Qwen3-Embedding-4B is a strong default (quality/latency balance); 0.6B for high-throughput/edge.

**Gotchas.** Instruction-prefix mismatch between indexing and query time silently degrades recall; MRL min dim is 32; quantized GGUF community variants exist but validate recall. HF code examples set `max_length=8192` (a demo default, not the 32K model limit).

**Getting started.** Serve 4B on vLLM with `--runner pooling`; index documents with no instruction; query with the `Instruct:` prefix; store in Qdrant with MRL truncation (see §5).

**Primary sources:** arxiv.org/abs/2506.05176 · huggingface.co/Qwen/Qwen3-Embedding-{0.6B,4B,8B} · github.com/QwenLM/Qwen3-Embedding · docs.vllm.ai/en/latest/models/pooling_models/embed/

---
### 3. CacheBlend / LMCache KV-cache reuse for RAG on vLLM

**The problem.** RAG concatenates multiple retrieved chunks; only a common *prefix* is reusable with standard prefix caching. Because retrieved chunks appear in varying positions and lack cross-attention to preceding text, precomputed per-chunk KV caches aren't directly reusable — so prefill is recomputed every query.

**CacheBlend mechanism (EuroSys 2025 Best Paper, arXiv:2405.16444).** Pre-compute each chunk's KV cache independently, then at serving time (1) reuse all chunks' KV regardless of position, and (2) selectively recompute the KV of a small subset of "cross-attention-deviant" tokens — the tokens whose attention most changes given the other chunks — to restore full-prefill quality. Per the CacheBlend paper (Yao et al., EuroSys 2025, UChicago/Microsoft), it has "minimal loss in quality compared with full KV recompute, with 5%–18% selective recompute ratio," and "reduces TTFT by 2.2~3.3× and increases throughput by 2.8~5× under negligible quality drop." The small recompute is pipelined with KV retrieval, so slower/cheaper storage tiers can hold the cache without adding latency. Claim: near-100% KV cache hit rate for RAG.

**LMCache (github.com/LMCache/LMCache).** Production KV-cache layer; standalone daemon (no fate-sharing with the engine — cache survives engine crashes). Multi-tier: GPU → CPU DRAM (hot) → local disk → remote shared backend. Integrates into vLLM v1 via the KV-connector API (`--kv-transfer-config` with `kv_connector='LMCacheConnectorV1'`); default chunking 256 tokens; maintains a token-sequence→KV index enabling cross-request and cross-instance hits.

**Reported performance.** LMCache with vLLM: 3–10× latency reduction vs recompute on the CPU tier; the LMCache paper (arXiv:2510.09665) reports "1.9 to 8.1× smaller TTFT" at QPS=1 and "2.3–14× higher query processing rate…than the strongest baseline across five evaluated models." The vLLM Production Stack advertises up to ~15× higher throughput for long shared prefixes. Concrete long-context example (Backend.AI, 2026): "When VAST Data tested vLLM and LMCache on a DGX SuperPOD environment, loading a precomputed KV cache cut TTFT at 128K context from over 11 seconds down to 1.5."

**Production readiness (2026).** LMCache "graduated to production in January 2026" and is used by Google Cloud GKE Inference, CoreWeave, and Cohere; the vLLM Production Stack integration ships in LMCache 0.4+. CacheBlend-style blending is available via LMCache.

**AKS integration.** Deploy via the vLLM Production Stack Helm chart; set `lmcacheConfig.enabled: true` and `cpuOffloadingBufferSize`; use a PVC for the disk tier; for multi-replica, run a remote cache server so requests routed to different pods share KV. Pair with KV-aware routing.

**Limitations.** Blending (non-prefix reuse) trades a small quality delta for speed — measure on your data; chunk ordering and positional handling affect results; quality impact must be validated per model. Alternatives: vLLM automatic prefix caching (simplest, prefix-only), RAGCache (arXiv:2404.12457), TurboRAG (2410.07590), EPIC/position-independent caching (2410.15332), CacheCraft, and newer CacheClip (2510.10129).

**Getting started.** Start with vLLM `--enable-prefix-caching` (free win for shared system prompts). Add LMCache CPU offload next. Enable CacheBlend blending only after measuring answer-quality parity.

**Primary sources:** arxiv.org/abs/2405.16444 · arxiv.org/abs/2510.09665 · github.com/LMCache/LMCache · docs.lmcache.ai · blog.lmcache.ai · docs.vllm.ai/projects/production-stack

---
### 4. ColPali / ColQwen2.5 vision retrieval

**Architecture.** ColPali (arXiv:2407.01449, ICLR 2025) applies ColBERT-style **late interaction** (MaxSim) over patch-level embeddings from a VLM, indexing document *page images* directly — no OCR/layout pipeline. Query text tokens are matched against page patch vectors via MaxSim.

**Benchmarks.** Evaluated on ViDoRe v1 (in-domain) and ViDoRe v2 (out-of-domain, arXiv:2505.17166). ColQwen2/ColQwen2.5 (Qwen2-VL/Qwen2.5-VL backbones) improve over the original PaliGemma-based ColPali; ColQwen2.5 is slower (~188ms/batch vs ~86ms for ColPali-3B per the ColMate paper) but stronger. 2026 successors: ColQwen3-4B (state-of-the-art on ViDoRe), ColModernVBERT (250M params, within 0.6 NDCG@5 of ColPali at 10× fewer params), ColNomic, Jina-embeddings-v4 multivector.

**Licensing (VERIFY PER CHECKPOINT — the main risk).**
- Original ColPali: LoRA adapters MIT, but the **PaliGemma backbone is under Google's Gemma license** (not OSI-permissive).
- `vidore/colqwen2.5-v0.2` card states: "ColQwen2.5's vision language backbone model (Qwen2.5-VL) is under **Qwen RESEARCH LICENSE AGREEMENT**… adapters under MIT."
- Other ColQwen2.5 community checkpoints (e.g., `yydxlv/colqwen2.5-7b`, `tsystems/colqwen2.5-3b-multilingual`, `Metric-AI/ColQwen2.5-7b-multilingual`) list the backbone as **Apache-2.0** with MIT adapters.
- **Net:** conflicting license tags exist across ColQwen2.5 checkpoints. For commercial permissive-only use, select a checkpoint whose specific Qwen2.5-VL backbone is genuinely Apache-2.0 (base Qwen2.5-VL-3B/7B-Instruct are Apache-2.0; the 72B is Qwen license). The training data (`vidore/colpali_train_set`) is CC-BY-NC-4.0 — matters if you retrain, not for inference.

**Storage math.** ColPali generates ~1,030 vectors per page (Qdrant: "ColPali generates 1,030 vectors for just one page of a PDF"; ColQwenX doubles this). At 128 dims × float32, that is ~1,030 × 128 × 4 ≈ 527 KB/page uncompressed — expensive at scale.

**Token pooling (Qdrant).** Mean-pool by grid rows to shrink first-stage vectors. Qdrant's measured two-stage result: pooled retrieve top-200 → rerank with full-resolution → top-20; **retrieval 13× faster** with mean pooling preserving **NDCG@20 = 0.952, Recall@20 = 0.917** vs original ColPali; max pooling underperformed. The Visual RAG Toolkit paper (arXiv:2602.12510) reports reducing ~1024 → ~32 vectors/page (row/col pooling = fixed 32× for ColPali) with ~4× throughput and negligible quality loss at k≤10, plus "token hygiene" (strip BOS/EOS/prompt/padding tokens that distort MaxSim).

**Two-stage retrieval in Qdrant.** Store pooled vectors and full multivectors as named vectors; use a prefetch on pooled vectors + exact MaxSim rerank on full embeddings, server-side in one Query API call.

**Serving.** Use colpali-engine (illuin-tech/colpali) for embeddings; generation over retrieved pages with Qwen2.5-VL on vLLM (native VLM support). Binary quantization reduces memory but not the number of MaxSim comparisons (only pooling does that).

**When it beats OCR+text.** Visually rich, figure/table/chart-heavy PDFs, forms, and multilingual scanned docs where OCR is lossy (DocVQA-style). For clean digital text, a strong text pipeline (Qwen3-Embedding + BM25) is cheaper and usually sufficient.

**Getting started.** Convert PDFs → page images → colpali-engine embeddings; index pooled + full multivectors in Qdrant; two-stage query; feed top pages to Qwen2.5-VL on vLLM. Confirm your chosen checkpoint's backbone license first.

**Primary sources:** arxiv.org/abs/2407.01449 · arxiv.org/abs/2505.17166 · github.com/illuin-tech/colpali · qdrant.tech/blog/colpali-qdrant-optimization/ · qdrant.tech/documentation/tutorials-search-engineering/pdf-retrieval-at-scale/ · huggingface.co/vidore/colqwen2.5-v0.2

---
### 5. MRL + int8 quantization + rescoring in Qdrant

**MRL (arXiv:2205.13147).** Matryoshka Representation Learning nests coarse-to-fine information in prefixes of one embedding, so truncating to the first *d* dims yields a usable lower-dim vector "at least as accurate as independently trained low-dimensional representations," with no extra inference cost. Qwen3-Embedding supports MRL truncation to as low as 32 dims.

**Qdrant scalar quantization (int8).** Converts float32→int8 (75% memory reduction per value), partially reversible; uses the 99% value range (via `quantile`, e.g., 0.99) to preserve accuracy. Qdrant keeps original float32 vectors (on disk) so it can rescore.

**Binary quantization.** 1 bit/dim (32× compression); works best on high-dimensional models (≥1024 dims). Qdrant docs: use binary quantization **only with rescoring enabled**; Cohere embed-english-v2.0 (4096d) hit 0.98 recall@50 with 2× oversampling. Qdrant 1.15+ supports asymmetric query encoding (`query_encoding='scalar8bits'`) — 1-bit storage + 8-bit query for better precision.

**Oversampling + rescore API.** Search params `quantization: {rescore: true, oversampling: 2.4}` pre-selects `limit × oversampling` candidates from the quantized index, then re-scores with original vectors. Rescoring is on by default for binary/TurboQuant methods; scalar does not rescore by default — enable it. Reported <2% recall loss with oversampling 3.0 for binary. `always_ram: true` keeps quantized vectors in RAM while the HNSW graph/originals can live on disk.

**Config recipe (collection).**
```json
{
  "vectors": {"size": 1024, "distance": "Cosine"},
  "quantization_config": {"scalar": {"type": "int8", "quantile": 0.99, "always_ram": true}},
  "hnsw_config": {"on_disk": true}
}
```
Query with `params: {quantization: {rescore: true, oversampling: 2.0}}`.

**Combining MRL + quantization.** First truncate (e.g., Qwen3-4B 2560→1024 via MRL), then int8-quantize the truncated vector, then rescore against the stored higher-precision (truncated) originals. Storage math: 1024-dim float32 = 4KB/vector → int8 = 1KB → binary = 128 bytes; MRL truncation from 2560→1024 is a further ~2.5× before quantization.

**Limitations.** Rescoring from on-disk originals slows queries under I/O pressure; binary quantization on low-dim vectors loses too much; over-aggressive MRL truncation degrades multilingual/Hungarian recall more than English — validate per language.

**Getting started.** Enable scalar int8 + rescore + oversampling 2.0 with `always_ram`; measure recall@k vs float baseline; only then experiment with MRL truncation and binary quantization.

**Primary sources:** arxiv.org/abs/2205.13147 · qdrant.tech/documentation/guides/quantization/ · qdrant.tech/articles/scalar-quantization/

---
### 6. LettuceDetect groundedness / hallucination detection

**How it works (arXiv:2502.17125).** Token-level classification on ModernBERT (long context up to 8k; the paper trains at 4k) over (context, question, answer) triples, flagging unsupported answer spans. Encoder-based, ~30× smaller than the best LLM judges. Authored by Ádám Kovács (Budapest University of Technology & Economics) & Gábor Recski (TU Wien).

**Numbers.** Per arXiv:2502.17125, `lettucedetect-large-v1` "achieves an overall F1 score of 79.22%, outperforming prompt-based methods like GPT-4 (63.4%) and encoder-based models like Luna (65.4%)… surpasses fine-tuned LLAMA-2-13B (78.7%)" on RAGTruth. Throughput: **30–60 samples/s on a single GPU**.

**v2 / multilingual.** GitHub (KRLabsOrg/LettuceDetect) v2 adds `lettucedect-v2-qwen-2b` (generative, typed spans) and `lettucedect-v2-mmbert-base` (fast encoder), covering code, tool output, and prose across **14-language PsiloQA including Hungarian** — a strong fit for your bilingual corpus.

**License.** MIT (paper CC-BY-4.0; code + models MIT).

**Integration (inline gate in LangGraph).** After generation, run LettuceDetect on (retrieved context, user question, draft answer). If unsupported spans exceed a threshold, branch to: (a) revise/regenerate constrained to supported content, (b) re-retrieve, or (c) abstain/flag. Because it returns token spans, you can surface the exact unsupported text to users or force citations for flagged spans.

**Limitations.** Weaker on code/tool-output in v1 (v2 targets this); trained primarily on the RAGTruth distribution; Hungarian performance via PsiloQA should be validated locally.

**Alternatives.** MiniCheck (arXiv:2404.10774; MiniCheck-FT5 770M reaches GPT-4-level at 400× lower cost; Bespoke-MiniCheck-7B is stronger but check its custom license); Lynx; Vectara HHEM-2.1 (open-source version available, English/French/German — no Hungarian). For permissive + Hungarian, LettuceDetect v2 is the best fit; MiniCheck-FT5 (English) is a solid second.

**Getting started.** `pip install lettucedetect`; load `lettucedect-v2-mmbert-base`; wire as a post-generation LangGraph node returning spans; set an abstention threshold on a labeled dev set.

**Primary sources:** arxiv.org/abs/2502.17125 · github.com/KRLabsOrg/LettuceDetect · huggingface.co/blog/adaamko/lettucedetect

---
### 7. Indirect prompt-injection defenses for RAG

**Threat model.** Your corpus may contain untrusted documents (poisoned corpus). OWASP LLM Top 10 2025 ranks Prompt Injection as **LLM01**. RAG-specific attacks: **PoisonedRAG** (arXiv:2402.07867, USENIX Security 2025) — inject a few malicious texts to force an attacker-chosen answer; **AgentPoison** (arXiv:2407.12784, NeurIPS 2024) — backdoor an agent's memory/RAG store via optimized triggers (requires white-box embedder access). Indirect injection can also exfiltrate data or trigger tools.

**Defenses (layered).**
- **Spotlighting / delimiting (Microsoft, arXiv:2403.14720):** transform untrusted input (delimiting, datamarking, encoding) to give the model a continuous provenance signal separating data from instructions.
- **Instruction hierarchy (arXiv:2404.13208):** train/prompt the model to prioritize privileged (system/developer) instructions over retrieved-content instructions.
- **Architectural isolation — CaMeL (Google DeepMind, arXiv:2503.18813):** extract control/data flow from the *trusted* query via a custom Python interpreter; untrusted retrieved data can never alter program flow; capability-based policies prevent exfiltration. Reported: solves 67% (v1) / 77% (v2) of AgentDojo tasks with provable security vs 84% undefended (no guarantees). Dual-LLM patterns are a lighter-weight cousin.
- **Detection classifiers:** Meta PromptGuard 2 (Llama-licensed — NOT permissive, flag); ProtectAI `deberta-v3-base-prompt-injection-v2` (verify Apache-2.0 on the card). Meta SecAlign (arXiv:2507.02735) is a secure-foundation approach.
- **Ingestion-time sanitization:** strip/neutralize instruction-like content, HTML/hidden text, zero-width chars at index time; CommandSans (arXiv:2510.08829) does surgical prompt sanitization.
- **Canary tokens** to detect exfiltration; output filtering.

**Evaluation.** InjecAgent and AgentDojo (NeurIPS 2024) benchmarks for agentic injection.

**Practical layered architecture (your stack).** (1) Ingestion sanitizer before Qdrant/OpenSearch/Neo4j writes; (2) spotlighting/delimiting of retrieved chunks in the generation prompt; (3) instruction hierarchy in system prompt; (4) a permissive detection classifier (ProtectAI DeBERTa) as a LangGraph guard node on both query and retrieved content; (5) CaMeL-style capability control on tool calls in LangGraph (untrusted data cannot parameterize dangerous tools); (6) canary tokens + output scanning.

**Limitations.** No single defense is complete; adaptive attackers bypass fine-tuning-based defenses (arXiv:2507.07417) and even layered prompting under domain-camouflage (arXiv:2606.18530). CaMeL degrades utility. Treat defense-in-depth as risk reduction, not elimination.

**Getting started.** Implement ingestion sanitization + spotlighting first (cheap, high value); add ProtectAI DeBERTa guard node; scope LangGraph tool capabilities so retrieved content can't trigger side effects.

**Primary sources:** genai.owasp.org/llmrisk/llm01-prompt-injection/ · arxiv.org/abs/2402.07867 · arxiv.org/abs/2407.12784 · arxiv.org/abs/2403.14720 · arxiv.org/abs/2503.18813 · arxiv.org/abs/2404.13208 · huggingface.co/protectai/deberta-v3-base-prompt-injection-v2

---
### 8. miniCOIL / SPLADE++ learned sparse retrieval

**How learned sparse works.** SPLADE (v2 arXiv:2109.10086; v3 arXiv:2403.06789) uses the MLM head to produce a sparse term-weight vector over the vocabulary with **term expansion** (adds related terms not in the text) + learned weighting, controlled by FLOPS regularization for sparsity; scored via inner product on an inverted index. SPLADE-v3 is "statistically significantly more effective than both BM25 and SPLADE++," >40 MRR@10 on MS MARCO dev, +2% BEIR out-of-domain.

**miniCOIL (Qdrant).** A lightweight sparse neural retriever built on the BM25 formula: keeps BM25's exact term matching but attaches a per-token 4D contextual vector so the same surface term is weighted by meaning (e.g., "apple slicer" ranks "apple cutter" over "apple charger"). Unlike SPLADE it does **not** expand to new terms. Requires the IDF modifier enabled in Qdrant. Model: `Qdrant/minicoil-v1` via FastEmbed; Qdrant's current recommendation for new sparse projects. (Also `Qdrant/bm42-all-minilm-l6-v2-attentions`.)

**OpenSearch neural sparse (Apache-2.0).** Two modes: **bi-encoder** (encode query + doc) and **doc-only** (encode doc at ingest, tokenizer-only at query = inference-free, faster). Models: `opensearch-neural-sparse-encoding-doc-v3-distill` and `-doc-v3-gte` (pruned at max-value-ratio 0.1 for smaller index) + `opensearch-neural-sparse-tokenizer-v1`; bi-encoder `-v2-distill`. All Apache-2.0.

**Multilingual / Hungarian (the key constraint).** Most SPLADE/neural-sparse models are English-only. OpenSearch released its **first multilingual neural sparse model** `opensearch-neural-sparse-encoding-multilingual-v1` (Apache-2.0, 15 languages, MIRACL-evaluated, arXiv:2411.04403) — verify Hungarian is in its 15. The most reliable Hungarian learned-sparse option is **BGE-M3's sparse weights** (produces dense+sparse+multivector, broad language coverage). English-only SPLADE-v3 is fine for the English half of your corpus but not Hungarian.

**Latency / index-size trade-offs.** Learned sparse expansion inflates postings vs BM25 (larger index, higher query cost); doc-only/inference-free modes and pruning mitigate. miniCOIL stays close to BM25 index size since it doesn't expand terms.

**Hybrid integration (your stack).** Run BM25 (baseline) and/or a learned-sparse model in OpenSearch, dense (Qwen3-Embedding) in Qdrant; fuse via RRF; then bge-reranker-v2-m3. In Qdrant, a prefetch runs sparse + another dense, fused server-side.

**Getting started.** Keep BM25 as baseline; add OpenSearch multilingual-v1 (doc-only) for Hungarian+English or BGE-M3 sparse; measure nDCG uplift vs BM25 before committing; only adopt if the learned model clearly beats BM25 on your eval.

**Primary sources:** arxiv.org/abs/2109.10086 · arxiv.org/abs/2403.06789 · qdrant.tech/articles/hybrid-search/ · qdrant.tech/course/essentials/day-3/sparse-retrieval-demo/ · docs.opensearch.org/latest/ml-commons-plugin/pretrained-models/ · opensearch.org/blog/advancing-search-with-opensearch-v3-neural-sparse-models-and-a-multilingual-retrieval-model/ · huggingface.co/opensearch-project/opensearch-neural-sparse-encoding-multilingual-v1

---
### 9. Provence context pruning + lost-in-the-middle mitigation + citations

**Provence (ICLR 2025, arXiv:2501.16214).** A DeBERTa-v3-based model that **unifies reranking + sentence-level context pruning**: two heads — a pruning head (token-level binary mask, thresholded ~0.1–0.5) and a reranking head (BOS-token relevance score, MSE-distilled from a DeBERTa reranker). Cross-encodes query+passage, dynamically selects how much to prune per context, robust across domains. Trained on MS MARCO (3.2M docs) + Natural Questions with a SPLADE-v3 + DeBERTa-v3 pipeline. Speeds generation and cuts context noise plug-and-play for any LLM.

**LICENSE — FLAG.** The official weights `naver/provence-reranker-debertav3-v1` are **CC-BY-NC-4.0 (non-commercial)** per the HF blog and independent analyses ("not allowed for commercial use"). (Note: one HF README snapshot shows `license: cc-by-4.0`, but the model card's License section and Naver's blog state CC-BY-NC-4.0 — treat as **non-commercial until Naver clarifies**; do not deploy commercially.) **Permissive alternatives:** train your own extractive pruner on a permissive DeBERTa/ModernBERT (the paper shows the standalone pruner is a simple BERT + pruning head on LLM-generated targets), use RECOMP-style extractive pruning, or use your bge-reranker-v2-m3 (permissive) for reranking + a permissive sentence-level relevance classifier for pruning.

**Lost-in-the-middle (Liu et al., arXiv:2307.03172, TACL 2024).** LLMs use information best at the **beginning or end** of the context; performance degrades significantly for relevant info in the middle, even in long-context models. Mitigations: (1) **reorder** retrieved chunks so the most relevant are first/last (fold-in ordering); (2) query-aware contextualization (query before AND after the docs) — helps key-value retrieval but little for multi-doc QA; (3) attention-sorted reordering (Peysakhovich & Lerer) using models' preferential attention; (4) aggressively prune (Provence-style) so less lands in the middle; (5) keep top-k modest (Anthropic found 20 > 10 > 5 for their setup, but more chunks distract — tune).

**Fine-grained citations (self-hosted).** Emulate Anthropic's Citations API pattern: (a) assign stable chunk IDs; (b) prompt the model to cite chunk IDs inline; (c) quote-first generation (extract supporting quotes before composing the answer); (d) verify citations post-hoc with LettuceDetect (§6). Context formatting: XML-style delimiters (`<document id=..>`) tend to be parsed more reliably than markdown for context blocks and pair well with spotlighting (§7).

**Getting started.** Add relevance-ordered chunk placement (most-relevant first/last) immediately (free). Add extractive pruning (permissive model) before generation. Emit chunk-ID citations + verify with LettuceDetect.

**Primary sources:** arxiv.org/abs/2501.16214 · huggingface.co/blog/nadiinchi/provence · huggingface.co/naver/provence-reranker-debertav3-v1 · arxiv.org/abs/2307.03172 · aclanthology.org/2024.tacl-1.9/

---
### 10. Parent-child / auto-merging chunking + RAPTOR

**Parent-document / small-to-big.** Embed small child chunks for precise retrieval, but return the larger parent chunk (or a surrounding window) to the LLM for context. LangChain **ParentDocumentRetriever** (e.g., 100-token children → 500-token parents); LlamaIndex **sentence-window** (retrieve a sentence, expand to a window). Practitioner reports cite 15–30% accuracy gains on context-dependent queries (directional, not peer-reviewed).

**Auto-merging (LlamaIndex AutoMergingRetriever).** Build a HierarchicalNodeParser tree (coarse→fine). At query time, retrieve leaf nodes; if enough leaves under the same parent are retrieved (beyond a threshold), **merge them into the parent node** to give the LLM consolidated context instead of fragments.

**RAPTOR (arXiv:2401.18059, ICLR 2024).** Recursively embed (SBERT) → reduce dims (UMAP, global then local) → **soft-cluster (Gaussian Mixture Models)** → summarize each cluster with an LLM → repeat up the tree, producing multi-level summaries. Retrieval: **collapsed tree** (flatten all nodes at all levels into one pool and retrieve by cosine similarity) outperforms tree-traversal. Per the RAPTOR paper (Sarthi et al., Stanford): "by coupling RAPTOR retrieval with the use of GPT-4, we can improve the best performance on the QuALITY benchmark by 20% in absolute accuracy"; SOTA on multi-step reasoning QA.

**Cost & incremental-update problem.** Building the tree costs one LLM summarization pass per cluster per level (non-trivial for large corpora). **RAPTOR trees do not update incrementally** — adding documents ideally requires re-clustering/re-summarizing. Strategies: rebuild per-document subtrees, batch periodic rebuilds, or segment trees per document/collection so updates are localized.

**Open-source impls.** Original `parthsarthi03/raptor`; LlamaIndex RaptorPack; RAGFlow integration.

**When RAPTOR helps vs flat.** Use for holistic/summary/"what are the themes" and multi-hop questions needing cross-document synthesis; flat retrieval suffices for fact-lookup/extractive questions. Run both and route by query type.

**Combining with Neo4j (your stack).** Model the RAPTOR tree as graph nodes: leaf chunks and summary nodes as `:Chunk`/`:Summary` nodes, `PARENT_OF`/`CHILD_OF` edges, cluster membership as edges. Store embeddings in Qdrant keyed by Neo4j node IDs. This unifies auto-merging (walk up `PARENT_OF`) and RAPTOR (retrieve summary nodes) into graph traversals, and lets you localize tree rebuilds to affected subgraphs on document updates. License: RAPTOR paper CC-BY-4.0; original repo permissive; LlamaIndex/LangChain MIT.

**Getting started.** Start with LangChain ParentDocumentRetriever (cheap, big win). Add RAPTOR per-document subtrees for holistic questions, stored as Neo4j nodes + Qdrant vectors; use collapsed-tree retrieval; schedule batch rebuilds for updates.

**Primary sources:** arxiv.org/abs/2401.18059 · github.com/parthsarthi03/raptor · docs.llamaindex.ai/en/stable/examples/retrievers/auto_merging_retriever/ · python.langchain.com (ParentDocumentRetriever)

---
## Consolidated link list by technique

**1. Contextual Retrieval** — anthropic.com/engineering/contextual-retrieval · platform.claude.com/cookbook/capabilities-contextual-embeddings-guide · docs.together.ai/docs/how-to-implement-contextual-rag-from-anthropic

**2. Qwen3-Embedding** — arxiv.org/abs/2506.05176 · huggingface.co/Qwen/Qwen3-Embedding-0.6B · …-4B · …-8B · github.com/QwenLM/Qwen3-Embedding · docs.vllm.ai/en/latest/models/pooling_models/embed/

**3. CacheBlend / LMCache** — arxiv.org/abs/2405.16444 · arxiv.org/abs/2510.09665 · github.com/LMCache/LMCache · docs.lmcache.ai · blog.lmcache.ai · docs.vllm.ai/projects/production-stack

**4. ColPali / ColQwen2.5** — arxiv.org/abs/2407.01449 · arxiv.org/abs/2505.17166 (ViDoRe v2) · arxiv.org/abs/2602.12510 (Visual RAG Toolkit) · github.com/illuin-tech/colpali · qdrant.tech/blog/colpali-qdrant-optimization/ · qdrant.tech/documentation/tutorials-search-engineering/pdf-retrieval-at-scale/ · huggingface.co/vidore/colqwen2.5-v0.2

**5. MRL + Quantization in Qdrant** — arxiv.org/abs/2205.13147 · qdrant.tech/documentation/guides/quantization/ · qdrant.tech/articles/scalar-quantization/

**6. LettuceDetect** — arxiv.org/abs/2502.17125 · github.com/KRLabsOrg/LettuceDetect · huggingface.co/blog/adaamko/lettucedetect · (alts: arxiv.org/abs/2404.10774 MiniCheck · vectara.com HHEM-2.1)

**7. Prompt-injection defenses** — genai.owasp.org/llmrisk/llm01-prompt-injection/ · arxiv.org/abs/2402.07867 (PoisonedRAG) · arxiv.org/abs/2407.12784 (AgentPoison) · arxiv.org/abs/2403.14720 (Spotlighting) · arxiv.org/abs/2503.18813 (CaMeL) · arxiv.org/abs/2404.13208 (Instruction Hierarchy) · huggingface.co/protectai/deberta-v3-base-prompt-injection-v2

**8. Learned sparse** — arxiv.org/abs/2109.10086 (SPLADE v2) · arxiv.org/abs/2403.06789 (SPLADE-v3) · qdrant.tech/articles/hybrid-search/ · docs.opensearch.org/latest/ml-commons-plugin/pretrained-models/ · opensearch.org/blog/advancing-search-with-opensearch-v3-neural-sparse-models-and-a-multilingual-retrieval-model/ · huggingface.co/opensearch-project/opensearch-neural-sparse-encoding-multilingual-v1

**9. Provence + lost-in-the-middle** — arxiv.org/abs/2501.16214 · huggingface.co/blog/nadiinchi/provence · huggingface.co/naver/provence-reranker-debertav3-v1 · arxiv.org/abs/2307.03172 · aclanthology.org/2024.tacl-1.9/

**10. Parent-child + RAPTOR** — arxiv.org/abs/2401.18059 · github.com/parthsarthi03/raptor · docs.llamaindex.ai/en/stable/examples/retrievers/auto_merging_retriever/

---
## Recommendations (staged implementation order with dependency notes)

**Stage 0 — Foundations (do first, no dependencies).**
1. **Qwen3-Embedding-4B on vLLM** (§2) — Apache-2.0, Hungarian+English, replaces/augments your embedder. Prereq for §1 and §5.
2. **MRL + int8 quantization + rescoring in Qdrant** (§5) — depends on the embedder; enable scalar int8 + oversampling first, defer binary/MRL truncation until recall is measured.
3. **vLLM automatic prefix caching** (§3, first step) — free latency win, prereq for LMCache.

**Stage 1 — Retrieval quality.**
4. **Anthropic Contextual Retrieval, self-hosted** (§1) — depends on Stage 0 embedder + a vLLM small model + prefix caching. Highest retrieval-quality ROI.
5. **Learned sparse / hybrid** (§8) — keep BM25 baseline; add OpenSearch multilingual-v1 or BGE-M3 sparse for Hungarian; fuse with dense + bge-reranker-v2-m3.
6. **Lost-in-the-middle ordering + extractive pruning** (§9) — reordering is free; use a permissive pruner (NOT Provence weights commercially).

**Stage 2 — Serving efficiency & structure.**
7. **LMCache KV-cache reuse** (§3) — depends on stable vLLM deployment; add CPU offload, then CacheBlend blending after quality validation.
8. **Parent-child / RAPTOR + Neo4j** (§10) — depends on embedder + Neo4j; start with parent-document, add RAPTOR for holistic queries.

**Stage 3 — Safety & trust (wrap everything).**
9. **Prompt-injection defenses** (§7) — ingestion sanitization + spotlighting + instruction hierarchy + permissive detection classifier + LangGraph capability control.
10. **LettuceDetect groundedness gate** (§6) — MIT, Hungarian-capable; post-generation LangGraph node with abstention threshold.

**Optional / conditional.**
11. **ColPali/ColQwen2.5 vision retrieval** (§4) — only if you have visually rich/scanned PDFs; verify backbone license per checkpoint; use Qdrant two-stage pooled retrieval.

**Benchmarks/thresholds that change the plan.** If local 1−recall@20 doesn't improve ≥20% from Contextual Retrieval, drop it (cost isn't justified). If binary quantization loses >2% recall@k after rescoring, stay on int8. If a learned-sparse model doesn't beat BM25 on your nDCG@10 eval, keep BM25. If RAPTOR doesn't help holistic-question accuracy in A/B, restrict to flat retrieval. If LMCache CacheBlend blending shifts answer-quality metrics beyond your tolerance, use prefix-only caching.

## Caveats
- **License risks to flag:** Provence weights (CC-BY-NC-4.0, non-commercial); PaliGemma/Gemma-licensed ColPali; ColQwen2.5 backbone license varies by checkpoint (Qwen Research License vs Apache-2.0); Meta PromptGuard 2 (Llama license); ColPali training data CC-BY-NC-4.0; Bespoke-MiniCheck-7B (check its custom license). Verify every model card at deploy time — cards change.
- **Vendor vs independent numbers:** Anthropic's 35/49/67%, Qwen's leaderboard rank, and the LMCache/CacheBlend/Qdrant speedups are largely vendor-reported on their own setups (the CacheBlend TTFT/throughput figures and LMCache TTFT figures are from the authors' papers). Re-measure on your bilingual corpus and hardware.
- **Hungarian coverage is under-documented:** Qwen3-Embedding lists Hungarian but publishes no per-language score; learned-sparse Hungarian options are limited to multilingual models/BGE-M3; LettuceDetect Hungarian rides on PsiloQA. Validate all three on a local Hungarian eval set before relying on them.
- **Forward-looking items:** ColQwen3-4B, ColModernVBERT, LettuceDetect v2, and various 2026 cache systems (CacheClip, EPIC) are recent; treat maturity claims cautiously and pin versions.
- **A note on some 2026-dated arXiv IDs** encountered in reference lists (e.g., CacheClip 2510.10129, Visual RAG Toolkit 2602.12510): these appear in citing papers' bibliographies; confirm each preprint independently before building on it, as very recent IDs can be unstable or pre-publication.