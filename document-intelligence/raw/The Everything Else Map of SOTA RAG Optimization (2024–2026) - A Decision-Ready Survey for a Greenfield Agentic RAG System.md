# The "Everything Else" Map of SOTA RAG Optimization (2024–2026): A Decision-Ready Survey for a Greenfield Agentic RAG System

## TL;DR
- The single highest-ROI additions for your self-hosted AKS/vLLM/Neo4j+Qdrant+OpenSearch bilingual stack are: **Anthropic-style contextual retrieval at ingestion** (Anthropic's Sept 2024 post reports contextual embeddings + contextual BM25 cut top-20 retrieval failure rate 49% (5.7%→2.9%), and 67% (5.7%→1.9%) when reranking is added), **Qwen3-Embedding (Apache-2.0) to replace/augment BGE-M3** (the 8B model ranked #1 on the MTEB multilingual leaderboard at score 70.58 as of June 5, 2025, well ahead of BGE-M3's 54.6 retrieval), **ColPali/ColQwen2.5 vision retrieval over your rendered page images** (you already run layout detection, so this is a natural extension for tables/charts/scans), **LettuceDetect groundedness checking** (MIT, ModernBERT, 14-language including Hungarian, ~30× smaller than LLM judges), and **CacheBlend/LMCache KV-cache reuse on vLLM** (2.2–3.3× lower TTFT and 2.8–5× higher throughput for RAG).
- Adopt-now, low-risk building blocks you likely lack: **Matryoshka + int8/binary quantization with float rescoring** in Qdrant (up to ~80% storage cut with near-parity recall), **learned sparse retrieval (miniCOIL/SPLADE++) inside your existing hybrid**, **context pruning (Provence) instead of heavier LLMLingua**, **RAPTOR hierarchical summaries + parent-child/auto-merging chunking**, and **indirect prompt-injection defenses** because your corpus may contain untrusted documents.
- Treat as extension points (build the seam, defer the model): LLM/reasoning listwise rerankers (RankZephyr/Rank-R1/Qwen3-Reranker), DSPy/MIPROv2 automated prompt optimization, RAFT/Search-R1-style trained retrieval policies, and temporal-KG agent memory (Zep/Graphiti). Skip for now: full generator fine-tuning, binary quantization at 1-bit without rescoring, and graph-heavy memory variants whose cost rarely justifies the thin quality gain.

## Key Findings

**1. Vision retrieval is the biggest architectural opportunity you haven't tapped.** Because you already run a Visual Document Understanding pipeline with layout detection, adding ColPali/ColQwen2.5 late-interaction retrieval over rendered page images is a low-friction, high-upside extension. It bypasses OCR brittleness for tables, charts, and scanned pages, and Qdrant natively supports the required multi-vector (MaxSim) representation. The catch is storage: ColPali emits ~1,030 vectors/page, so you must use token pooling + a two-stage retrieve-then-rerank pattern.

**2. Your embedding choice is now outdated.** BGE-M3 is a solid MIT-licensed baseline, but Qwen3-Embedding (Apache-2.0, 0.6B/4B/8B, 32k context, 100+ languages, MRL + instruction-aware) is materially stronger on multilingual retrieval and is the right default for a Hungarian+English system.

**3. Ingestion-time enrichment beats most retrieval-time tricks per dollar.** Contextual retrieval, late chunking, RAPTOR, and parent-child chunking all attack the root cause of RAG failure — context-poor chunks — and compound with everything downstream.

**4. Generation/serving economics are where you'll win latency and cost.** KV-cache reuse (CacheBlend), context pruning (Provence), quantization, and semantic answer caching are the levers; long-context "just stuff it" is not a substitute for retrieval at your scale.

**5. Reliability is non-negotiable for a bilingual, possibly-untrusted corpus.** LettuceDetect (Hungarian-relevant, from KRLabs/BME authors) for groundedness plus indirect-prompt-injection defenses belong in v1, not v2.

---

## Details by Pipeline Stage

### Stage 1 — Ingestion & Contextual Enrichment

**Anthropic Contextual Retrieval (ADOPT NOW).** Prepend an LLM-generated, chunk-specific context blurb to each chunk *before* embedding and *before* building the BM25 index. Per Anthropic's Sept 2024 engineering post "Contextual Retrieval in AI Systems": Contextual Embeddings alone cut top-20 retrieval failure rate 35% (5.7%→3.7%); + Contextual BM25 → 49% (5.7%→2.9%); reranked Contextual Embedding + Contextual BM25 reduced it 67% (5.7%→1.9%). Economics hinge on prompt caching: you generate the context for every chunk once at ingestion, and prompt caching of the parent document makes this cheap. For your stack, run this with a small model on vLLM at ingestion. Verdict: **adopt** — highest single ROI, and it directly strengthens your existing hybrid BM25+dense+RRF+reranker chain.

**Late Chunking (Jina) (ADOPT / evaluate).** Instead of embedding chunks independently, embed the *whole* document through a long-context model, then mean-pool token embeddings per chunk. This keeps pronouns/references/thematic context intact. Requires a long-context embedder (BGE-M3 and Qwen3-Embedding qualify at 8k/32k). It is cheaper than contextual retrieval (no LLM calls) but only captures *within-document* context. Verdict: **adopt as complement** — pair late chunking (cheap, within-doc) with contextual retrieval (LLM, adds cross-doc facts) where budget allows.

**RAPTOR hierarchical summarization (ADOPT for long docs).** Recursively cluster + summarize chunks into a tree; retrieve from both leaf chunks and higher-level summaries. Per Sarthi et al., RAPTOR (ICLR 2024, arXiv:2401.18059): "by coupling RAPTOR retrieval with the use of GPT-4, we can improve the best performance on the QuALITY benchmark by 20% in absolute accuracy." Integrated in RAGFlow. Complements your GraphRAG for multi-hop/holistic questions. Verdict: **adopt selectively** for long structured documents; it adds ingestion compute and a summary index.

**Parent-child / small-to-big / sentence-window / auto-merging chunking (ADOPT).** Index small chunks for precision but return the parent (or auto-merge contiguous children) for generation context. Low complexity, robust gains, orthogonal to everything else. Verdict: **adopt** — standard modern default; you likely have the markdown structure to make parent-child trivial.

**Metadata & synthetic-question enrichment (HyQE / doc2query) (EXTENSION POINT).** Generate titles/keywords/synthetic questions per chunk and index them to bridge the query-document vocabulary gap. Doc2query-style expansion is the "document-side" analog of query expansion you already do. Verdict: **extension point** — cheap to add later; measure against contextual retrieval first (overlapping benefit).

**Proposition-based indexing (SKIP/experimental).** Decompose text into atomic factoid "propositions" as retrieval units. Improves precision but multiplies index size and ingestion LLM cost, and interacts awkwardly with citation-to-source. Verdict: **skip** for v1; revisit if precision plateaus.

### Stage 2 — Embedding Optimization

**Qwen3-Embedding (ADOPT — replace/augment BGE-M3).** Apache-2.0; sizes 0.6B/4B/8B; 32k context; 100+ languages; instruction-aware (task prefixes give +1–5%); MRL (truncatable dims). Per QwenLM (arXiv:2506.05176, Zhang et al.): "The 8B size embedding model ranks No.1 in the MTEB multilingual leaderboard (as of June 5, 2025, score 70.58)," surpassing Gemini-Embedding. On MMTEB, Qwen3-Embedding-0.6B scores 64.3 Mean / 64.6 Retrieval vs multilingual-e5-large-instruct 63.22 / 57.1 and BGE-M3 59.56 / 54.6; Qwen3-Embedding-4B reaches 69.5 Mean / 72.3 Retrieval. For Hungarian specifically there is **no dedicated MTEB split** — Hungarian is covered inside MMTEB aggregate and tasks like WikipediaRetrievalMultilingual/BelebeleRetrieval. An older (2024, pre-Qwen3) Towards Data Science test on an EU-AI-Act Hungarian Q&A corpus found BGE-M3 > multilingual-e5-large among open models and flagged Hungarian as high-variance. Verdict: **adopt Qwen3-Embedding-4B (or 0.6B for cost)** as primary dense encoder; keep BGE-M3 available for its native multi-vector/sparse modes and as a cheap fallback. Run a Hungarian eval on your own data before committing (no public Hungarian leaderboard to lean on).

**Matryoshka Representation Learning (MRL) (ADOPT).** Truncate embedding dims (e.g., 1536→256) with near-zero recall loss. Uber Eats reported truncating gte-Qwen2 1536D→256D cost only 0.002 pp in R@200 while cutting storage to ~16% and index cost to ~23%. Qwen3-Embedding supports MRL natively. Verdict: **adopt** — set your Qdrant dim to the smallest MRL cut that passes eval.

**Embedding quantization (int8/binary) + float rescoring (ADOPT int8; binary with care).** int8 scalar quantization = 4× storage cut, and per Qdrant's default per-dimension int8 it "preserved cosine to three decimal places"; binary = 32× cut but shows "significant accuracy degradation" alone and needs a float-rescoring second pass to recover accuracy. The canonical pattern (traceable to the Binary Passage Retriever): retrieve candidates with compressed codes, rescore top-k with full-precision vectors. Combining MRL + quantization can reach ~80% cost reduction. Verdict: **adopt int8 by default in Qdrant; adopt binary + rescoring only for the largest collections after measuring the recall cliff.**

**Domain fine-tuning / hard-negative mining / LoRA embedders (EXTENSION POINT).** Fine-tuning embeddings on your domain (contrastive, with synthetic query generation and hard-negative mining) yields the biggest gains where public models are weak — plausibly your Hungarian domain corpus. Cost: a training pipeline + eval harness. Verdict: **extension point** — build the synthetic-data + hard-negative pipeline once your eval shows a ceiling with off-the-shelf Qwen3.

### Stage 3 — Storage & Serving

**HNSW tuning + on-disk + quantization in Qdrant (ADOPT).** Tune `m` and `ef_construct` for the recall/latency/build-time trade-off; enable `on_disk: true` for large collections; layer scalar/binary quantization. For ColPali multi-vectors, HNSW build cost grows quadratically with vectors/page, so mean-pool to a handful of vectors for first-stage HNSW and rerank with full multi-vectors (Qdrant reports mean pooling preserved NDCG@20 = 0.952 / Recall@20 = 0.917, near-identical to full ColPali, with 13× faster results). Verdict: **adopt** as standard operating procedure.

**DiskANN/Vamana & GPU indexing (CAGRA) (EXTENSION POINT).** Vamana/DiskANN gives billion-scale SSD-resident indexes (RAM-frugal, higher latency ~5–20ms p50) via a Vamana graph + product quantization; CAGRA builds indexes on GPU 10–50× faster than CPU HNSW but must fit in VRAM (practical to ~50M vectors/GPU). NVIDIA cuVS brings 40×+ Vamana build speedups and 8× faster Weaviate builds. Qdrant uses CPU HNSW; Milvus 2.5 supports both CAGRA and DiskANN natively; pgvector now has two DiskANN implementations. Verdict: **extension point** — only if you outgrow Qdrant's in-memory HNSW; consider Milvus only at very large scale.

**Single-source-of-truth storage (ADOPT pattern).** Store originals in object storage, chunks+metadata in a system of record, vectors in Qdrant, sparse in OpenSearch, graph in Neo4j — with a stable `doc_id`/`chunk_id` join key and content hashes for cache/index invalidation. Incremental/streaming indexing keyed on content hash avoids full rebuilds. Verdict: **adopt** — this is the backbone that makes freshness, ACLs, and citations tractable.

### Stage 4 — Query & Retrieval (beyond what you have)

**Learned sparse retrieval: miniCOIL / SPLADE++ (ADOPT miniCOIL).** These replace/augment BM25 with context-aware term weights. Qdrant's own **miniCOIL** (their current recommendation for new projects) keeps BM25's exact-term matching but reweights terms by contextual meaning (ranks "apple cutter" over "apple charger" for "apple slicer"); it requires the IDF modifier in Qdrant. SPLADE++ (SPLADE-v3 line) adds learned term *expansion* and is the strongest classic learned-sparse model. Verdict: **adopt miniCOIL in your Qdrant sparse leg** (or SPLADE++/neural-sparse in OpenSearch), then A/B against plain BM25 on Hungarian data — sparse-neural multilingual behavior is uneven (Qdrant explicitly recommends experimenting on your data), so measure.

**Late interaction as first-stage / multi-vector reranking (ADOPT as reranker).** Beyond ColPali-for-vision, ColBERT-style late interaction (MaxSim) is a strong *text* reranker and can be a first-stage retriever (PLAID/WARP) at scale. GTE-ModernColBERT is a current text option. Verdict: **adopt late interaction as a tensor reranker** where your cross-encoder budget is tight; first-stage PLAID is an extension point.

**Temporal/recency-aware retrieval (ADOPT if corpus is time-sensitive).** Boost by recency/validity windows; store `valid_from/valid_to` metadata. Verdict: **adopt** if your documents supersede each other (policies, regulations).

**Text-to-SQL / table-aware retrieval (EXTENSION POINT).** For structured/tabular data, expose text-to-SQL as a LangGraph tool (parallel to your Cypher tools) and use table-aware chunking/QA. Verdict: **extension point** — natural fit given you already parameterize Cypher; add when tabular questions appear.

### Stage 5 — Reranking (beyond cross-encoders)

**LLM listwise / reasoning rerankers (EXTENSION POINT).** RankZephyr (7B, GPT-4-distilled listwise) and reasoning rerankers Rank1 (pointwise, R1-distilled), Rank-R1 (setwise, GRPO-trained), Rank-K, ReasonRank, and FIRST (single-token-decoding for speed) beat cross-encoders on reasoning-heavy benchmarks (BRIGHT, R2MED) but cost 1–2 orders of magnitude more latency (listwise is O(n) with sliding-window/tournament sort). Qwen3-Reranker (Apache-2.0, 0.6B/4B/8B) is the strongest open-weight option and beats bge-reranker-v2-m3 on multilingual reranking. Verdict: **keep bge-reranker-v2-m3 as default; make the reranker a swappable interface** and route only hard/ambiguous queries (via your Adaptive-RAG complexity router) to an LLM reranker. Consider Qwen3-Reranker for a unified multilingual upgrade.

### Stage 6 — Context Assembly & Generation

**Context pruning: Provence (ADOPT) over LLMLingua (evaluate).** Provence (ICLR 2025) is a DeBERTa-based context pruner purpose-built for RAG that removes a large fraction of context (~49% in their tests) with minimal quality loss and negligible latency — it is an encoder, unlike LLM-based LongLLMLingua which is the slowest pruner. LongLLMLingua boosts NaturalQuestions by up to 21.4% at ~4× compression and achieves up to 94% cost reduction on LooGLE, but is question-aware (no context caching) and doubles compute vs LLMLingua. Verdict: **adopt Provence** as a lightweight pruner between rerank and generation; use LongLLMLingua only if you need extreme token compression and can absorb the latency.

**Lost-in-the-middle mitigation & context ordering (ADOPT).** Place the most relevant chunks at the head and tail, not the middle; budget the context window explicitly per stage; format context with clear structural delimiters (markdown/XML) and stable per-chunk IDs to enable citations. Verdict: **adopt** — free win, pure prompt-assembly logic.

**Citation grounding / quote extraction (ADOPT).** Require the generator to emit chunk-ID citations and, ideally, verbatim supporting quotes; this pairs with your verification layer. Verdict: **adopt.**

**Long-context vs RAG (DECISION).** Evidence is consistent: for large corpora, retrieval into a 32k window beats stuffing the full 128k. On LongBench v2, Qwen2.5 and GLM-4-Plus performed *better* at 32k retrieval context than using the full 128k without RAG (Qwen2.5 +4.1%); only GPT-4o leveraged 128k well, and still lagged its no-RAG score (−0.6%). Anthropic's own guidance: if your knowledge base is <200k tokens (~500 pages) you can skip RAG entirely and just prompt-cache the corpus. Verdict: **keep RAG as primary; add a "small-corpus / whole-doc stuffing" fast path** for tiny document sets, gated by your Adaptive-RAG router.

### Stage 7 — Serving Optimization (vLLM)

**KV-cache reuse for RAG: CacheBlend / LMCache (ADOPT).** vLLM's automatic prefix caching only helps when the *prefix* matches — in RAG the retrieved chunks change per query, so prefix caching mostly helps the first chunk. CacheBlend (Best Paper, EuroSys '25, arXiv:2405.16444) precomputes each chunk's KV cache independently and selectively recomputes a small fraction of tokens; per the paper it "reduces time-to-first-token (TTFT) by 2.2-3.3× and increases the inference throughput by 2.8-5× from full KV recompute without compromising generation quality." LMCache integrates CacheBlend with vLLM and reports up to 4.5× end-to-end speedup with a one-line change. Verdict: **adopt** — this is the biggest RAG-specific serving win and directly targets your vLLM deployment. Note: chunk-order/formatting affects reuse (see CacheWeaver), so keep chunk boundaries stable.

**Speculative decoding (EXTENSION POINT).** Speeds token generation on vLLM generally; not RAG-specific. Verdict: **extension point** — enable if generation latency dominates.

**Semantic answer caching: GPTCache-style (ADOPT with guardrails).** Cache full RAG answers keyed by query embedding similarity; GPTSemCache reports hit rates of 61.6–68.8% and up to 68.8% reduction in API calls with high accuracy. Critical risks: (1) no sharp invalidation boundary — a static high-similarity threshold can serve wrong answers; (2) embedding-collision adversarial hits (SAFE-CACHE research); (3) semantic drift over time. Mitigations: two-tier (static curated Q&A with long TTL + dynamic with aggressive eviction), content-hash-based invalidation tied to your source-of-truth, and grounded cache routing that re-verifies. Verdict: **adopt a conservative two-tier semantic cache**; never cache across ACL boundaries.

### Stage 8 — Verification & Reliability

**Groundedness / hallucination detection: LettuceDetect (ADOPT — Hungarian-relevant).** MIT-licensed, ModernBERT-based token-level detector from Kovács & Recski (KRLabs/BME). Per arXiv:2502.17125, it achieves "an F1 score of 79.22% for example-level detection, which is a 14.8% improvement over Luna, the previous state-of-the-art encoder-based architecture," while being "approximately 30 times smaller than the best models" and processing 30–60 examples/sec on a single GPU with up to 8k context. The v2 line is multilingual (14-language PsiloQA incl. Hungarian) and now covers code/tool output. Alternatives: MiniCheck, Lynx (NLI/entailment checkers). Verdict: **adopt LettuceDetect** as an inline groundedness gate — Hungarian-relevant authorship makes it exactly right for your bilingual corpus. Note: an independent 2026 benchmark found LettuceDetect-large weak on *code-agent* hallucinations (0.17 span-F1), so scope it to prose/document QA.

**RAG security — indirect prompt injection (ADOPT DEFENSES).** OWASP LLM01:2025 ranks prompt injection #1; RAG/fine-tuning do not eliminate it. Indirect injection via poisoned retrieved docs is the guaranteed attack for any corpus that ingests untrusted content. Per Chen et al., AgentPoison (NeurIPS 2024, arXiv:2407.12784): "AgentPoison achieves an average attack success rate of ≥ 80% with minimal impact on benign performance (≤ 1%) with a poison rate < 0.1%," and even a single poisoning instance with a single-token trigger reaches ≥60% ASR, transferable across model families. Layered defenses: (1) **architectural isolation** of retrieved data from the instruction channel (most durable, independent of model); (2) input spotlighting/delimiting of retrieved content (SpotLight); (3) hybrid search + reranking raise the injection bar (attacker must beat both lexical and semantic stages — you already have this); (4) knowledge-base sanitization (CleanBase-style malicious-doc detection at ingestion); (5) classifiers (PromptGuard/deberta-v3-prompt-injection) on retrieved chunks. Verdict: **adopt architectural isolation + spotlighting + ingestion sanitization** in v1.

**PII handling & ACL-aware retrieval (ADOPT).** Filter documents by access-control metadata *at retrieval time* (per-user/tenant/namespace) before they reach the LLM; never let semantic cache or prefix cache cross tenant boundaries (vLLM supports per-request cache salting for isolation). PII detection/redaction at ingestion. Verdict: **adopt** — bake document-level security filters into every retrieval call.

**Observability & cost engineering (ADOPT).** Instrument with OpenTelemetry GenAI semantic conventions; use Langfuse or Arize Phoenix for tracing/eval. Track token budgets per stage and use small-model cascades (cheap model first, escalate on low confidence — aligns with your Adaptive-RAG router). Verdict: **adopt** from day one.

### Stage 9 — Training-Side / Prompt-Side (mostly extension points)

**DSPy / MIPROv2 automated prompt optimization (EXTENSION POINT — high value).** MIPROv2 uses Bayesian optimization over instructions + few-shot demos; integrated in RAGAS and DeepEval. Newer GEPA reports roughly a 13% aggregate improvement over MIPROv2 with ~35× fewer rollouts on its benchmarks. Cost: needs a labeled eval set and is compute-heavy (full MIPROv2 optimization can require ~1,000 LLM calls for the instruction-proposal step). Verdict: **extension point** — once your RAGAS/DeepEval harness has a stable dataset, run MIPROv2/GEPA on your generation and rewrite prompts; expect meaningful, cheap-at-inference gains.

**RAFT — retrieval-aware generator fine-tuning (EXTENSION POINT).** Train the generator on gold + distractor contexts with CoT + verbatim citations; RAFT reports gains over domain-SFT+RAG of +30.87% on HotpotQA and +31.41% on the HuggingFace dataset. Risk: RAFT-trained models over-answer even on all-noise context (mitigate with "I don't know" alignment, e.g., Divide-Then-Align). ALoFTRAG/CRAFT do this with LoRA on in-domain unlabeled data (privacy-friendly). Verdict: **extension point** — worth it only after prompt-only + retrieval quality plateau; LoRA keeps it affordable on vLLM.

**RL retrieval policies — Search-R1 / DeepRetrieval / ReSearch (EXTENSION POINT / watch).** Search-R1 (Qwen2.5) reports +41% over RAG baselines (7B) and +20% (3B) across seven QA datasets — but that headline is vs *plain* RAG; vs the strongest non-RL baseline the paper tests, the margin is ~+24%. DeepRetrieval trains query rewriting via RL against a live search engine with a small model. Verdict: **extension point** — your LangGraph agentic loop is the right substrate, but treat these as research-grade; the RL training cost is high and gains are benchmark-dependent.

### Stage 10 — Agent Memory (complement to RAG)

**Mem0 vs Zep/Graphiti (EXTENSION POINT).** For persistent agent memory (distinct from document RAG): Zep/Graphiti is a temporal knowledge graph (bi-temporal fact validity) — an independent LongMemEval evaluation measured Zep at 63.8% vs Mem0's 49.0% (GPT-4o); Zep's own paper reports 94.8% vs MemGPT's 93.4% on the DMR benchmark. But Mem0's own paper shows the graph variant (Mem0g) rarely justifies cost (68.44 vs 66.88 on LOCOMO, ~3× slower search, ~2× tokens), and Letta's "filesystem" approach scored 74.0% on LOCOMO, above Mem0g's 68.5%. Graphiti is Apache-2.0 (Zep's only open-source component as of April 2025) and can sit on your Neo4j. Verdict: **extension point** — you already have Neo4j; if you need cross-session user/agent memory, Graphiti-on-Neo4j is the natural fit, but don't over-invest in graph memory until benchmarks on your workload justify it.

---

## Prioritized Shortlist: Top 10 Highest-ROI Additions

1. **Contextual Retrieval at ingestion** (contextual embeddings + contextual BM25 with prompt caching). 49% failure-rate reduction (67% with reranking); strengthens every downstream stage. *Effort: medium (ingestion LLM pass). Risk: low.*
2. **Swap primary dense encoder to Qwen3-Embedding (4B or 0.6B), Apache-2.0.** SOTA multilingual (70.58 MMTEB for 8B), 32k context, MRL, instruction-aware. *Effort: low. Risk: low — but run a Hungarian eval on your data.*
3. **CacheBlend/LMCache KV-cache reuse on vLLM.** 2.2–3.3× lower TTFT, 2.8–5× throughput (up to 4.5× end-to-end via LMCache). *Effort: medium (LMCache integration). Risk: low.*
4. **ColPali/ColQwen2.5 vision retrieval** over your rendered page images, in Qdrant multi-vector with token pooling + two-stage rerank. Best for tables/charts/scans. *Effort: medium-high. Risk: medium (storage).*
5. **MRL + int8 quantization + float rescoring in Qdrant.** Up to ~80% storage/cost cut at near-parity recall. *Effort: low. Risk: low.*
6. **LettuceDetect groundedness gate** (MIT, Hungarian-relevant, ~30× smaller than LLM judges). *Effort: low. Risk: low.*
7. **Indirect prompt-injection defenses** (architectural isolation + spotlighting + ingestion sanitization + your existing hybrid+rerank). *Effort: medium. Risk: high if omitted (>80% ASR at <0.1% poison rate).*
8. **miniCOIL/SPLADE++ learned sparse leg** inside your hybrid, A/B vs BM25 on Hungarian. *Effort: low-medium. Risk: low.*
9. **Provence context pruning + lost-in-the-middle ordering + citations.** Cheaper tokens, better grounding. *Effort: low. Risk: low.*
10. **Parent-child/auto-merging chunking + RAPTOR summaries** for long docs (+20% abs. on QuALITY). *Effort: medium. Risk: low.*

Extension seams to build now but defer the model: swappable reranker interface (for RankZephyr/Qwen3-Reranker/Rank-R1), DSPy/MIPROv2 prompt-optimization harness, ACL-aware retrieval filter, two-tier semantic answer cache, and Graphiti-on-Neo4j agent memory.

---

## Updated End-to-End Reference Architecture

```
                            ┌─────────────────────── INGESTION ───────────────────────┐
  PDF ──► your PDF→MD/OCR ──► Visual Doc Understanding (layout detect) ──┐
                                                                          │
        ┌── TEXT PATH ──────────────────────────────────┐    ┌── VISION PATH (NEW) ──────────┐
        │ markdown header + recursive chunking           │    │ render page images            │
        │ + parent-child / auto-merging (NEW)            │    │ ColPali/ColQwen2.5 multi-vec  │
        │ + Contextual Retrieval blurb per chunk (NEW)   │    │ token pooling (NEW)           │
        │ + late chunking (NEW, complement)              │    │                               │
        │ + RAPTOR summaries for long docs (NEW)         │    └───────────────┬───────────────┘
        │ + metadata/synthetic-Q enrichment (ext)        │                    │
        │ + NER (GLiNER2/huBERT) + PII redaction (NEW)   │                    │
        │ + KB sanitization / injection scan (NEW)       │                    │
        └───────────────┬────────────────────────────────┘                   │
                        │                                                     │
     ┌──────────────────▼─────────────────────────────────────────────────────▼──────────────────┐
     │                                    STORAGE (source-of-truth pattern)                         │
     │  Object store: originals+page images │ Qdrant: dense (Qwen3-Embedding, MRL+int8, on-disk),   │
     │  multi-vector (ColPali) │ OpenSearch: sparse (BM25 + miniCOIL/SPLADE++, Contextual BM25) │    │
     │  Neo4j: document graph + Graphiti temporal memory (ext) │ stable doc_id/chunk_id + hashes     │
     └──────────────────┬───────────────────────────────────────────────────────────────────────┘
                        │
   QUERY ──► Adaptive-RAG complexity router ──► [simple | multi-hop decomp/IRCoT/step-back | web fallback]
                        │  query rewrite/expansion/HyDE/CSQE/multi-query (existing)
                        ▼
     ┌───────────────────────── RETRIEVAL (LangGraph orchestration) ─────────────────────────┐
     │  Hybrid: dense (Qdrant) + sparse (OpenSearch) + [vision multi-vec] + [Cypher/text-SQL]  │
     │  ──► RRF fusion (existing)  ──► ACL-aware filter (NEW)  ──► recency boost (opt)          │
     └───────────────────────────────────────┬───────────────────────────────────────────────┘
                                              │
   RERANK: bge-reranker-v2-m3 (default) │ swappable → Qwen3-Reranker / RankZephyr / Rank-R1 (ext, routed)
   [CRAG/Self-RAG corrective loop — existing]
                                              │
   CONTEXT ASSEMBLY: Provence pruning (NEW) → lost-in-the-middle ordering (NEW) → structured
   formatting + chunk-ID citations (NEW) → context-window budgeting
                                              │
   GENERATION: vLLM (Qwen2.5-VL for page-image answers) + CacheBlend/LMCache KV reuse (NEW)
   + speculative decoding (ext) │ grounded system prompt + abstention calibration
                                              │
   VERIFICATION: LettuceDetect groundedness gate (NEW) → citation/quote check → retry/abstain
                                              │
   SERVING/OPS: two-tier semantic answer cache (NEW, ACL-scoped) │ OpenTelemetry GenAI +
   Langfuse/Phoenix │ per-stage token budgets + small-model cascade
                                              │
   OFFLINE LOOP: RAGAS/DeepEval (existing) → DSPy/MIPROv2 prompt optimization (ext) →
   embedding fine-tune / RAFT / RL retrieval policy (ext)
```

---

## Trade-off Tables

### Ingestion enrichment
| Technique | Benefit | Cost | Verdict |
|---|---|---|---|
| Contextual Retrieval | −49% retrieval failures (−67% w/ rerank) | LLM pass at ingestion (prompt-cache offsets) | Adopt |
| Late chunking | Preserves within-doc context, no LLM | Long-context embed pass | Adopt (complement) |
| RAPTOR | +20% abs. on QuALITY (w/ GPT-4) | Recursive summarization compute | Adopt (long docs) |
| Parent-child/auto-merge | Precision + context, low complexity | Minimal | Adopt |
| Proposition indexing | Higher precision | Index bloat, citation friction | Skip v1 |

### Embedding & storage
| Technique | Benefit | Cost/Risk | Verdict |
|---|---|---|---|
| Qwen3-Embedding | SOTA multilingual (70.58 MMTEB, 8B) | Re-embed corpus; VRAM (4B/8B) | Adopt |
| MRL dim truncation | ~84% storage cut, ~0 recall loss | Requires MRL model | Adopt |
| int8 quantization + rescore | 4× storage, cosine preserved to 3 dp | Rescoring pass | Adopt |
| Binary quantization + rescore | 32× storage | Accuracy cliff without rescore | Adopt (large only) |
| DiskANN/CAGRA | Billion-scale / 10–50× GPU build | Not in Qdrant; VRAM (CAGRA) | Extension |

### Retrieval & rerank
| Technique | Benefit | Cost/Risk | Verdict |
|---|---|---|---|
| miniCOIL/SPLADE++ | Context-aware sparse > BM25 | Multilingual variance | Adopt (A/B) |
| ColPali vision | OCR-free tables/charts/scans | ~1000 vec/page storage | Adopt |
| Late-interaction reranker | Strong quality, fast | Multi-vector storage | Adopt (as reranker) |
| LLM/reasoning rerankers | Best on reasoning queries | 10–100× latency | Extension (routed) |

### Generation & serving
| Technique | Benefit | Cost/Risk | Verdict |
|---|---|---|---|
| CacheBlend/LMCache | 2.2–3.3× lower TTFT, 2.8–5× throughput | Integration; stable chunking | Adopt |
| Provence pruning | ~49% context cut, fast (encoder) | Small quality risk | Adopt |
| LongLLMLingua | up to 21.4% gain, up to 94% cost cut | Slow, no caching | Evaluate only |
| Semantic answer cache | 61–69% hit rate | Stale/adversarial/ACL risk | Adopt (two-tier) |
| Long-context stuffing | Simple for tiny corpora | Worse than RAG at scale | Fast-path only |

### Reliability
| Technique | Benefit | Cost/Risk | Verdict |
|---|---|---|---|
| LettuceDetect | 79.22% RAGTruth F1, cheap, Hungarian | Weak on code (0.17 span-F1) | Adopt (prose) |
| Injection defenses | Blocks #1 OWASP risk (>80% ASR) | Some latency | Adopt |
| ACL-aware retrieval | Data-leak prevention | Metadata plumbing | Adopt |
| OTel + Langfuse/Phoenix | Debuggability, cost control | Setup | Adopt |

---

## Recommendations (staged)

**Phase 1 (v1 — do before launch):** Contextual Retrieval + parent-child chunking at ingestion; migrate dense encoder to Qwen3-Embedding-4B (validate on Hungarian data first); MRL + int8 quantization in Qdrant; add miniCOIL to the sparse leg; Provence pruning + lost-in-the-middle ordering + chunk-ID citations; LettuceDetect groundedness gate; indirect-injection defenses (architectural isolation + spotlighting + ingestion sanitization) + ACL-aware retrieval; OTel/Langfuse observability; CacheBlend/LMCache on vLLM.

**Phase 2 (post-launch, driven by eval):** ColPali/ColQwen2.5 vision path (trigger: measurable OCR failures on tables/charts/scans); RAPTOR for long docs; two-tier semantic answer cache; swappable LLM reranker routed via Adaptive-RAG; DSPy/MIPROv2 prompt optimization once the eval set is stable.

**Phase 3 (only if ceilings hit):** domain embedding fine-tuning with hard-negative mining; RAFT/LoRA generator adaptation; RL retrieval policies (Search-R1-style); Graphiti-on-Neo4j agent memory; DiskANN/CAGRA or Milvus at very large scale.

**Benchmarks that change these decisions:** If your Hungarian retrieval recall with Qwen3-Embedding beats BGE-M3 by <2 points on your own eval, keep BGE-M3 (MIT, native sparse+multi-vector). If binary quantization drops recall >3–5 pp even with rescoring, stay on int8. If OCR quality on your VDU pipeline yields >~95% correct answers on table/chart questions, defer ColPali. If semantic-cache false-hit rate exceeds your tolerance, tighten thresholds or restrict to the static curated tier. If LLM-reranker latency pushes p95 past your SLA, keep it off the default path.

## Caveats
- **No public Hungarian embedding leaderboard exists.** All Hungarian guidance rests on MMTEB *aggregate* multilingual scores plus one small pre-Qwen3 study; you must run your own Hungarian eval before committing to Qwen3-Embedding over BGE-M3. There is likewise no Hungarian-specific reranker — bge-reranker-v2-m3 (Apache-2.0) and Qwen3-Reranker (Apache-2.0) handle Hungarian only via multilingual coverage.
- **Licenses (all commercially usable and permissive):** Qwen3-Embedding & Qwen3-Reranker = Apache-2.0; BGE-M3 & multilingual-e5-large = MIT; bge-reranker-v2-m3 = Apache-2.0 (but bge-reranker-v2-gemma variants carry the Gemma license); LettuceDetect = MIT; Graphiti = Apache-2.0. EmbeddingGemma/gemini-embedding = CC-BY (Gemma-based); NV-Embed-v2 = non-commercial — avoid the last two on license grounds.
- Several cited arXiv identifiers surfaced in search carry 2026 dates (follow-on cache/embedding/security preprints); I relied on established primary results (CacheBlend EuroSys'25, LettuceDetect, Anthropic, Qwen3, RAPTOR, AgentPoison) and treated very recent preprints as directional.
- Vendor-reported numbers (Anthropic's failure-rate reductions, LMCache's 4.5×, Qdrant's ColPali speedups) are from the technique authors/vendors and should be reproduced on your data.
- Search-R1's "+41%" and similar RL headlines are measured against weak baselines; real-world gains vs a strong tuned RAG are smaller (~+24% in the same paper).
- Benchmark scores (MMTEB, RAGTruth, LongMemEval, LOCOMO, QuALITY) are proxies; retrieval quality is corpus-specific — treat every number here as a hypothesis to validate with RAGAS/DeepEval on your bilingual corpus.