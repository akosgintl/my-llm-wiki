# Query Modification for a SOTA Agentic RAG System: Taxonomy, Benchmarks, and a Staged Design

## TL;DR
- **Put a cheap, deterministic query-understanding layer on the fast path (normalization + conversational rewrite + asymmetric per-leg processing + Self-Query metadata extraction), and reserve expensive transformations (decomposition, step-back, HyDE-as-fallback) for the agentic path only** — this matches the evidence that light rewriting reliably helps while heavy expansion often hurts when your retriever and corpus are already strong.
- **Skip global HyDE/Query2Doc and always-on RAG-Fusion.** Multiple 2025-2026 studies show LLM expansion degrades retrieval when the retriever is strong, the domain is specialized/entity-heavy, or the LLM lacks corpus knowledge; RAG-Fusion recall gains are largely neutralized after reranking+truncation in production. Use CSQE/PRF-style corpus-grounded expansion instead, gated by the CRAG relevance grader.
- **For Hungarian, rely on BGE-M3 multilingual embeddings (best-evidenced open model for Hungarian), not query translation.** Cross-lingual evidence shows dense multilingual encoders derive little-to-no benefit from translation and translation adds noise + latency; huBERT-based Hungarian embeddings underperform BGE-M3.

## Key Findings

1. **Query rewriting reliably helps, and small models suffice.** Rewrite-Retrieve-Read (Ma et al., EMNLP 2023) trains a 770M T5-large rewriter via RL, "achieving an 82.2% hit rate on AmbigNQ compared to 76.4% for standard retrieval," and the small rewriter "often matched or outperformed a black-box LLM rewriter (ChatGPT) on tasks like HotpotQA and AmbigNQ." Conversational rewriting (condensing chat history to a standalone query) is the single highest-ROI transformation for multi-turn chat.

2. **LLM query expansion (HyDE, Query2Doc) is corpus- and retriever-dependent and frequently hurts.** Query2Doc boosts BM25 by 3-15% on MS-MARCO/TREC DL but only ~0.4-4% on already-strong dense retrievers. HyDE was designed for the *no-relevance-labels* regime; the original paper notes fine-tuned encoders gain little and can degrade. 2025-2026 work confirms HyDE/expansion fails on unfamiliar, ambiguous, entity-heavy, and domain-mismatched queries.

3. **Decomposition and step-back are for multi-hop/complex queries only.** Question decomposition yields large multi-hop gains ("gains in retrieval (MRR@10: +36.7%) and answer accuracy (F1: +11.6%) over standard RAG baselines" on MultiHop-RAG/HotpotQA) but can hurt single-hop/precise queries. Step-back prompting "improves PaLM-2L performance on MMLU (Physics and Chemistry) by 7% and 11% respectively, TimeQA by 27%, and MuSiQue by 7%." Both belong on the agentic branch.

4. **For Hungarian, multilingual embeddings win.** In the only dedicated Hungarian retrieval study (Antal, *Infocommunications Journal* 2025), "BGE-M3 and XLM-ROBERTA achieved the highest accuracy (MRR: 0.90) on the Clearservice dataset," while huBERT was explicitly listed among the "Weakest models." Translation-based pipelines don't beat dense multilingual encoders.

## Details

### 1. Query rewriting / reformulation
Rewrite-Retrieve-Read (Ma et al., 2023, EMNLP; arXiv:2305.14283) introduced the paradigm: a small trainable rewriter (T5-large, 770M) is warmed up on LLM pseudo-data then fine-tuned with PPO using reader accuracy as reward. It raised retrieval hit rate on AmbigNQ from 76.4% (standard retrieval) to 82.2%, and the 770M rewriter often matched or outperformed a black-box ChatGPT rewriter on HotpotQA and AmbigNQ — a resource-efficiency result central to your design. Successors: RQ-RAG (Chan et al., 2024) trains an end-to-end model for rewriting/decomposition/clarification; MaFeRw (AAAI 2025, arXiv:2408.17072) uses multi-aspect feedback; RAFE (arXiv:2405.14431) uses ranking feedback; a 2025 line (arXiv:2507.23242) uses annotation-free RL with verifiable search reward.

**Conversational query rewriting** condenses chat history into a standalone query. ChatQA (NVIDIA, arXiv:2401.10225) documents the problem: follow-ups with pronouns lack retrieval signal, but feeding full dialogue history is redundant. IBM's Granite 3.2 8B ships a LoRA "query rewrite" intrinsic specifically for decontextualizing the last user turn (arXiv:2504.11704). Best-practice prompts (SemEval-2026 Task 8 submissions) instruct: rewrite ONLY if context-dependent, otherwise return verbatim — blindly rewriting every query introduces noise.

**Small vs large rewriter cost/quality:** A 770M T5 rewriter matches ChatGPT on QA rewriting (Ma et al.). Practitioner evidence (ZeroEntropy) argues a 1B-class model fine-tuned on (intent → observed-recall) runs in single-digit ms and beats generic LLM rewrites on downstream recall. But small base models (<3B) can fail the task entirely: a context-understanding study (arXiv:2402.00858) found OPT-125M cannot complete rewriting (generates next sentence, rewrites wrong turn, or copies verbatim); ability emerges with scale. Implication: a 3-4B instruct model (Qwen 3 4B) on vLLM is the sweet spot for self-hosted rewriting.

### 2. Query expansion — and when it hurts
Classic PRF/RM3 draws expansion terms from top-retrieved documents. LLM-based expansion: **Query2Doc** (Wang et al., EMNLP 2023, arXiv:2303.07678) generates a pseudo-document and concatenates it — +3-15% BM25 on MS-MARCO/TREC DL, but only +0.4 to +4.0 on strong dense retrievers. **HyDE** (Gao et al., ACL 2023, arXiv:2212.10496) generates a hypothetical document, embeds it, and retrieves neighbors — designed for zero-shot/unsupervised dense retrieval without relevance labels.

**Critical evidence that expansion HURTS when retriever/corpus is strong or domain-mismatched:**
- HyDE's own authors: "HyDE with fine-tuned encoder is not the intended usage... less powerful instruction LMs can negatively impact the overall performance of the fine-tuned retriever." As the search log grows and the dense retriever strengthens, fewer queries should route to HyDE.
- "LLM-based Query Expansion Fails for Unfamiliar and Ambiguous Queries" (arXiv:2505.12694): expansion degrades when the LLM lacks knowledge of the query (introduces non-existent entities) and for ambiguous queries (biased expansions narrow coverage).
- Long-tail entity retrieval (Frontiers, 2026): "Despite its low per-query cost (247 tokens), HyDE degrades performance, resulting in negative return on investment" in entity-intensive retrieval.
- "Out of Style: RAG's Fragility to Linguistic Variation" (arXiv:2504.08231): HyDE+rerank improves robustness but "still lags behind original queries in performance."
- Enterprise practitioner consensus: synonyms that help consumer search are "lethal" in enterprise RAG ("term sheet" ≠ "contract"; "incident" ≠ "outage").

**Corpus-grounded expansion is safer:** **CSQE** (Lei et al., EACL 2024, arXiv:2402.18031) uses the LLM to extract pivotal sentences from initially-retrieved documents (PRF-style grounding) plus LLM expansions — "strong performance without any training, especially with queries for which LLMs lack knowledge." GenQREnsemble/GenQRFusion (Dhole & Agichtein, 2024) ensemble multiple keyword-expansion paraphrases. ThinkQE (2025) adds an iterative thinking phase with corpus feedback. A systematic PRF-with-LLMs study (arXiv:2603.11008) finds combining LLM + corpus feedback helps dense retrieval, best done by combining sources independently.

### 3. Multi-query / RAG-Fusion
RAG-Fusion (Rackauckas, arXiv:2402.03367) generates multiple query variants, retrieves for each, and fuses with Reciprocal Rank Fusion (RRF, k=60). DMQR-RAG (arXiv:2411.13154) adds diversity across rewrite strategies. Best reported: a "Hybrid+Diverse" config (BM25+vector × LLM rewrites, RRF, cross-encoder rerank) achieved +19% nDCG@10 and +18% MRR over baseline on the RAG-Fusion eval harness.

**But production evidence is sobering:** "Scaling RAG with RAG Fusion: Lessons from an Industry Deployment" (arXiv:2603.02153) finds fusion increases raw recall but "these gains are largely neutralized after re-ranking and truncation," benefits "highly context-dependent and largely confined to recall-scarce query regimes," and for most enterprise workloads fusion "introduces additional system cost without delivering material improvements." Implication: since this architecture already does hybrid BM25+dense with RRF and bge-reranker-v2-m3, adding multi-query LLM fusion on top yields diminishing returns.

### 4. Query decomposition for multi-hop
Self-Ask (Press et al., 2022), Decomposed Prompting (Khot et al., 2022), least-to-most, and IRCoT (Trivedi et al., ACL 2023, arXiv:2212.10509) established decomposition. IRCoT interleaves retrieval with each CoT step: up to +21 retrieval points and +15 QA points on HotpotQA/2WikiMultihopQA/MuSiQue/IIRC, and works with small models (Flan-T5-large). 2025 SOTA: "Question Decomposition for RAG" (Ammann, Golde & Akbik, ACL SRW 2025, arXiv:2507.00355) reports "gains in retrieval (MRR@10: +36.7%) and answer accuracy (F1: +11.6%) over standard RAG baselines" on MultiHop-RAG/HotpotQA — but warns "if a query is already precise, decomposition can introduce" noise, and "longer contexts can degrade downstream performance." "Query Decomposition for RAG: Balancing Exploration-Exploitation" (arXiv:2510.18633) adds adaptive per-subquery retrieval budgets. This directly maps to the agentic LangGraph branch (ReAct-style reason+retrieve loop).

### 5. Step-back / abstraction
Step-Back Prompting (Zheng et al., ICLR 2024, arXiv:2310.06117) elicits a higher-level abstraction question. Gains: "MMLU (Physics and Chemistry) by 7% and 11% respectively, TimeQA by 27%, and MuSiQue by 7%." Best for reasoning-intensive queries where the specific query is too narrow to retrieve governing principles. Belongs on the agentic path. Note GSM8K gains were minimal (84.3%, "on par with zero-shot CoT") — abstraction helps knowledge-intensive, not simple-arithmetic, tasks.

### 6. Intent classification & structured query construction
**Self-Query** (LangChain pattern) uses an LLM to extract metadata filters from natural language ("sci-fi movies after 2000 rated >8" → filter: genre=sci-fi, year>2000, rating>8) plus a semantic query. This maps directly to Neo4j parameterized Cypher and Qdrant payload filters. **Asymmetric per-leg processing:** extract keywords/entities for the BM25 (OpenSearch) leg and keep a natural-language/semantic query for the dense (BGE-M3/Qdrant) leg — the legs reward different query forms. NER (GLiNER2/huBERT) feeds both metadata filters and entity anchors that prevent semantic drift. NVIDIA's NeMo Retriever and Milvus support LLM-generated natural-language filters natively; the pattern is production-mature.

### 7. Spelling / multilingual / Hungarian
**Cross-lingual: prefer multilingual embeddings over translation.** "What Drives Cross-lingual Ranking?" (Goworek et al., arXiv:2511.19324, Nov 2025) finds dense CLIR models "derive little benefit from document translation" and translation "can even degrade slightly because of translation-induced noise"; it recommends "cross-lingual search systems should prioritise semantic multilingual embeddings and targeted learning-based alignment over translation-based pipelines, particularly for cross-script and under-resourced languages." (Caveat: it tests document-side translation and its language set excludes Hungarian; Finnish is the closest agglutinative proxy at ~67-68 nDCG@10 on MIRACL.)

**Hungarian specifics:** The only dedicated study (Antal, *Infocommunications Journal* 2025, DOI 10.36244/ICJ.2025.4.1) evaluated eight embedding models plus a BM25 baseline on two Hungarian datasets and found "BGE-M3 and XLM-ROBERTA achieved the highest accuracy (MRR: 0.90) on the Clearservice dataset, while GEMINI demonstrated superior performance on HuRTE (MRR: 0.99)"; huBERT was explicitly among the "Weakest models: GEMINI, HUBERT, MINILM" on the domain corpus. BGE-M3 (Chen et al., arXiv:2402.03216) is built on "XLM-RoBERTa-large with 550M parameters and a vocabulary of 250,000 subword tokens covering 105+ languages," unifies dense/sparse/multi-vector retrieval, and handles inputs "up to 8,192 tokens" — an ideal fit. MIRACL has no Hungarian split. Named Hungarian LLMs (PULI, Racka, SambaLingo-Hungarian) lack published retrieval benchmarks. BM25 was "competitive" on Hungarian domain data — validating the hybrid approach. Hungarian is morphologically rich (agglutinative), so the BM25 leg benefits from lemmatization/analyzer support and the multilingual dense leg carries most cross-lingual weight.

### 8. Where query modification sits in an adaptive router
The consensus is **hybrid**: a small amount of query understanding runs *pre-router* (normalization, conversational rewrite, language detection, intent/metadata extraction) because the router itself needs a clean standalone query to classify complexity; then *per-path* transformations run *post-router*. Adaptive-RAG (Jeong et al., NAACL 2024, arXiv:2403.14403) trains a T5-Large complexity classifier to route among no-retrieval / single-step / multi-step, matching always-expensive baselines at lower cost. Cache rewritten/normalized queries and HyDE generations (repeated queries otherwise double LLM cost).

### 9. Evaluation
Measure retrieval delta (nDCG@10, Recall@k, MRR) separately from end-to-end faithfulness/answer quality — a better-sounding answer can be grounded in worse evidence (Databricks guidance). A/B each transformation holding the rest of the pipeline fixed. Rewrite plausibility is a poor proxy for retrieval lift. Watch failure modes: query drift/semantic drift (expansion diverges from intent), over-expansion (excessively long queries), entity drift (generated "PostgreSQL 17 API" retrieves wrong version), and multi-turn drift (over-eager conversational rewrites).

## Updated Reference Architecture Data Flow

```
                         ┌─────────────────────────────────────────────┐
   User turn ───────────▶│  STAGE 0 — QUERY UNDERSTANDING (pre-router)  │
   + chat history        │  (Qwen 3 4B on vLLM + deterministic rules)   │
                         │  1. Normalize: unicode/case/spell/acronym    │
                         │     + domain-vocab map  (deterministic)      │
                         │  2. Language detect (HU / EN)                 │
                         │  3. Conversational rewrite → standalone q     │
                         │     ("rewrite only if context-dependent")    │
                         │  4. Self-Query: extract metadata filters      │
                         │     + NER entities (GLiNER2 / huBERT)         │
                         │  5. Asymmetric split:                         │
                         │     • keywords/entities → BM25 leg           │
                         │     • natural-language q → dense leg          │
                         │     • keep ORIGINAL q for reranking          │
                         │  [cache: session+turn keyed]                 │
                         └───────────────────────┬─────────────────────┘
                                                 ▼
                         ┌─────────────────────────────────────────────┐
                         │  ADAPTIVE COMPLEXITY ROUTER (Adaptive-RAG)   │
                         └───────┬──────────────────────────┬──────────┘
                    simple/single-pass                 complex/multi-hop
                                 ▼                              ▼
        ┌───────────────────────────────┐   ┌────────────────────────────────────┐
        │  FAST PATH (no expansion)     │   │  AGENTIC PATH (bounded LangGraph)   │
        │  • Hybrid retrieve:           │   │  • Decompose → sub-questions (≤N)   │
        │    OpenSearch BM25 +          │   │  • Step-back abstraction (if        │
        │    BGE-M3 dense (Qdrant)      │   │    reasoning-intensive)             │
        │    + Neo4j Cypher (filters)   │   │  • IRCoT reason+retrieve loop       │
        │  • RRF fusion                 │   │  • HyDE/Query2Doc = FALLBACK ONLY,  │
        │  • bge-reranker-v2-m3         │   │    prefer CSQE (corpus-grounded)    │
        └───────────────┬───────────────┘   │  • per-hop hybrid retrieve+RRF+rrk  │
                        │                    └──────────────────┬─────────────────┘
                        └──────────────┬────────────────────────┘
                                       ▼
                    ┌────────────────────────────────────────┐
                    │  CRAG RELEVANCE GRADER                  │
                    │  Correct → refine & generate            │
                    │  Ambiguous → merge KB + web             │
                    │  Incorrect → rewrite q + web fallback   │
                    │              (Tavily / SearXNG)          │
                    └───────────────────┬────────────────────┘
                                        ▼
                              Main LLM (vLLM) → grounded answer
```

Key insertion points: (a) Stage 0 sits **before** the router so the router classifies a clean, standalone query; (b) expensive transformations (decomposition, step-back, HyDE) live **only** on the agentic branch; (c) HyDE/expansion is a **grader-gated fallback**, never always-on.

## Recommendations

**Stage 0 — Fast path, pre-router (always on, cheap, deterministic-first):**
1. Normalization: Unicode/casing, spelling correction, acronym/domain-vocabulary mapping via a maintained dictionary (deterministic, no LLM).
2. Language detection (route Hungarian vs English; both handled by BGE-M3).
3. Conversational rewrite: Qwen 3 4B on vLLM, "rewrite only if context-dependent, else return verbatim." Cache by (session, turn).
4. Self-Query metadata extraction: extract Neo4j/Qdrant filters + entities (GLiNER2/huBERT NER). Cheap and high-precision-boosting.
5. Asymmetric per-leg: keyword/entity query → OpenSearch BM25; natural-language query → BGE-M3 dense. Keep original query for reranking.

**Stage 1 — Fast path retrieval:** hybrid BM25+dense → RRF → bge-reranker-v2-m3. Do NOT add HyDE or multi-query here.

**Stage 2 — Agentic path (complex queries, bounded LangGraph loop):**
- Query decomposition into sub-questions (bounded, e.g., ≤4 hops).
- Step-back abstraction when the query is reasoning-intensive/too specific.
- IRCoT-style interleaved retrieve+reason.
- HyDE / Query2Doc **as a fallback only**, triggered by the CRAG grader when relevance is low AND the query looks unfamiliar/vocabulary-mismatched — prefer CSQE (corpus-grounded) over vanilla HyDE. Cache generations.

**Stage 3 — CRAG grading + web fallback:** relevance grader routes Correct/Ambiguous/Incorrect; on Incorrect, rewrite query and fall back to Tavily/SearXNG.

**What to skip and why:** always-on HyDE/Query2Doc (hurts strong-retriever/entity-heavy/domain-mismatched queries); always-on RAG-Fusion multi-query (recall gains neutralized after your existing rerank+truncation); query translation for Hungarian (multilingual embeddings win, translation adds noise+latency); a Hungarian-only embedding model (huBERT underperforms BGE-M3).

**Staged rollout with benchmarks that change the plan:**
1. *Phase 1 (ship first):* Stage 0 deterministic normalization + conversational rewrite + asymmetric per-leg + Self-Query. Measure Recall@20/nDCG@10 vs. baseline. Threshold: keep any transformation that shows a positive retrieval delta with CI excluding zero; disable those that regress.
2. *Phase 2:* Wire Adaptive-RAG router + agentic decomposition/step-back on the complex branch. Threshold: enable step-back only for the reasoning-intensive intent class where it beats plain decomposition.
3. *Phase 3:* Add CRAG-gated CSQE/HyDE fallback. Threshold: enable HyDE only for the query segment where the grader flags low relevance AND the fallback improves faithfulness in A/B.
4. *Re-evaluate:* If retrieval Recall@20 is consistently low (recall-scarce regime), enable multi-query RAG-Fusion. If a dedicated Hungarian eval shows huBERT/PULI beating BGE-M3 on your corpus, revisit the embedding choice. If the fast-path rewriter shows query drift in A/B, tighten the "return verbatim" prompt or downgrade to deterministic-only.

## Trade-off Table

| Technique | Typical gain | Cost (latency/tokens) | Risk | When to use |
|---|---|---|---|---|
| Normalization/spelling/acronym | Small, reliable | Near-zero (deterministic) | Very low | Always, fast path |
| Conversational rewrite | Large in multi-turn | 1 small-LLM call | Over-rewriting/drift | Always in chat, pre-router |
| Self-Query metadata extraction | Precision boost | 1 small-LLM call | Wrong filters exclude docs | When corpus has rich metadata |
| Asymmetric per-leg | Moderate | Minimal | Low | Always with hybrid retrieval |
| Query2Doc/HyDE | +3-15% BM25; ~0 on strong dense | 1 large gen + embed | Entity/semantic drift; hurts strong retrievers | Fallback only, weak-retrieval/OOV queries |
| CSQE (corpus-grounded) | Robust, esp. OOV | Initial retrieval + LLM | Long queries | Preferred expansion, agentic fallback |
| Multi-query / RAG-Fusion | +18-19% nDCG isolated; ~0 after rerank | N× retrieval + N gens | Cost, drift | Recall-scarce regimes only |
| Decomposition | +36.7% MRR@10 multi-hop | N sub-queries + LLM | Hurts single-hop | Agentic path, multi-hop |
| Step-back | +7-27% reasoning tasks | 1 LLM call | Over-abstraction | Agentic path, reasoning-intensive |
| IRCoT interleaving | +15-21 pts multi-hop | Multiple LLM+retrieval | Latency, loops | Agentic path, bounded |

## Implementation Notes

**Small-model rewriter on vLLM:** Serve Qwen 3 4B (permissive Apache-2.0) with vLLM for conversational rewrite, Self-Query extraction, and decomposition. Reserve the main LLM for generation and the CRAG grader. Batch rewrite/extraction calls. Use guided/structured decoding (JSON) for Self-Query filters and decomposition lists. Note that sub-3B base models can fail rewriting outright (arXiv:2402.00858), so 3-4B instruct is the practical floor.

**Prompt patterns:**
- *Conversational rewrite* — "Rewrite ONLY the final user turn into a standalone query; if already standalone, return it EXACTLY; do not answer; do not add facts; output only the query."
- *Self-Query* — structured JSON output constrained to allowed filter fields and types (year, doc_type, section, entity); return `null` filter when none apply.
- *Step-back* — few-shot abstraction template (Zheng et al. Knowledge-QA template: "What are the principles/higher-level concept behind this question?").
- *Decomposition* — "Generate logically-ordered single-hop sub-questions needed to answer; stop at N; if the question is already single-hop, return it unchanged."

**Caching:** Cache normalized/rewritten queries (session-keyed), Self-Query filters, and any HyDE/CSQE generations (query-hash keyed). Caching HyDE generations is explicitly recommended to avoid doubling the LLM bill on repeated queries.

**Eval strategy:** Build a labeled eval set from your own corpus — the Hungarian study shows public benchmarks don't cover your domain, and Hungarian MTEB coverage is classification-only (no retrieval tasks). For each transformation, run an isolated A/B holding retrieval/rerank/generation fixed; report Recall@k and nDCG@10 (retrieval) plus faithfulness and answer correctness (end-to-end). Track per-transformation contribution and per-language (HU/EN) breakdowns. Gate each transformation behind a feature flag so you can disable any that regress. Use confidence intervals at realistic sample sizes (the RAG-Fusion harness shows many "gains" vanish under proper CIs).

## Caveats
- Several strong sources are practitioner blogs or preprints (arXiv not yet peer-reviewed); load-bearing claims here rest on peer-reviewed venues (EMNLP/ACL/NAACL/EACL/ICLR), with industry reports flagged separately.
- The Hungarian retrieval evidence rests substantially on a single study with a small (50-question) domain set (Clearservice); treat as directional and validate on your own corpus.
- arXiv:2511.19324 (cross-lingual) tests document-side translation and its language set excludes Hungarian; the "translate the query to English" scenario is under-tested for Hungarian specifically. Finnish (~67-68 nDCG@10 on MIRACL) is the closest agglutinative proxy.
- Benchmark gains are dataset-specific (open-domain QA, MS-MARCO, BEIR); your enterprise/Hungarian corpus may differ materially — the eval harness is non-optional.
- RAG-Fusion and expansion numbers vary widely by config; the "+19% nDCG" figure is from the technique's own eval harness and may be optimistic relative to your already-reranked pipeline.
- Some 2026-dated arXiv identifiers appeared in search results (e.g., production RAG-Fusion and PRF-with-LLM studies); these are recent and should be re-verified against final published versions before citing in formal documentation.