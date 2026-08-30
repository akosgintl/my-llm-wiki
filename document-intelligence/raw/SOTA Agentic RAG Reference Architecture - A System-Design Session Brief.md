# SOTA Agentic RAG Reference Architecture: A System-Design Session Brief

## TL;DR
- **Build a modular, "simple-first" hybrid RAG core** — BM25 + dense vector retrieval fused with Reciprocal Rank Fusion (RRF), then a cross-encoder reranker (bge-reranker-v2-m3, Apache-2.0) — and wrap it in an **adaptive router** that sends easy queries to a single-pass fast path and only escalates complex/multi-hop queries into an agentic loop. This matches the 2025-2026 consensus that multi-hop agentic retrieval delivers large gains on genuinely compositional questions (e.g., MuSiQue) but adds ~2-3× latency and cost for little gain on single-fact lookups.
- **Use Neo4j as a lightweight "document/lexical graph + entity graph," not a heavyweight ontology.** The reference paper you cited (Wedge et al., arXiv:2606.05901, "Reducing Hallucinations in Complex QA using Simple Graph-based RAG") shows a *simple* document graph (Document→Section→Paragraph with links) queried by pre-built Cypher tools more than halved refusals and hallucinations on the MoNaCo complex-QA benchmark with only a modest token increase. Keep vectors in a dedicated store (Qdrant, Apache-2.0) alongside Neo4j, joined by shared IDs, rather than relying solely on Neo4j's built-in vector index.
- **Keep ingestion decoupled from retrieval behind clean interfaces** so your future PDF→Markdown/OCR pipeline drops in without touching the query path. Make NER, GraphRAG, and web search pluggable extension points. For Hungarian, standardize on BGE-M3 or multilingual-e5-large-instruct embeddings and huBERT/HuSpaCy NER; use LangGraph for orchestration and RAGAS/DeepEval for an eval-driven loop.

## Key Findings

### The reference paper (arXiv:2606.05901)
Wedge, Stutter, Dixon & Cała (National Innovation Centre for Data, Newcastle University; v2 dated 22 Jul 2026) argue that "it is now typically expected that RAG systems should be agentic for all but the simplest queries." Their central design choice is deliberately *lightweight*: rather than an LLM-constructed rich knowledge graph, they build a **simple document graph** (document titles, section titles, constituent paragraphs, with links between paragraphs/documents) in Neo4j, and expose the agent to **pre-written Cypher-backed tools** rather than having the LLM generate raw Cypher — explicitly to "relieve the agent from the task of generating valid data queries – a potential source of failure and security vulnerability." Evaluated on MoNaCo (1,315 human-written complex questions), their vector+graph RAG, compared with vector-only RAG and zero-shot: "more than halved the proportion of complex questions that the agent refused to answer," increased precision/recall of factual correctness, "can halve the number of hallucinated answers," and achieved the highest fine-grained truthfulness score — "all with only a modest increase in token usage." They use LLM-as-a-judge metrics (Answer Relevancy, Faithfulness, Context Precision, Context Recall). This is strong evidence for your instinct: start simple, add a lightweight graph as a structure-aware retrieval enabler, and prefer curated graph tools over free-form text2cypher for reliability/security. (Note this is a preprint licensed CC BY-NC-SA 4.0; the authors say the evaluation KG will be released on publication.)

### Hybrid retrieval and fusion
- **RRF is the right default fusion method.** It operates on ranks, not raw scores, so it needs no score-scale calibration between BM25 and cosine similarity — the architectural mistake most failed production hybrid implementations share is trying to combine BM25 and cosine with a single weighted-score formula. The original method (Cormack, Clarke & Büttcher, "Reciprocal Rank Fusion outperforms Condorcet and Individual Rank Learning Methods," SIGIR 2009, DOI 10.1145/1571941.1572114) uses a default k=60 and "consistently yields better results than any individual system, and better results than the standard method Condorcet Fuse." Production systems tune k (OpenSearch added native RRF in 2.19/Feb 2025; Weaviate and Qdrant also ship it natively). Empirically, RRF gives consistent but modest gains over the best single retriever (e.g., +0.0094 nDCG@100 on the DAPFAM patent benchmark; >40% Recall@10 gains over BM25-only on a scientific-code benchmark).
- **The correct architecture is two-stage:** first-stage hybrid retrieval (BM25 ∪ dense) fused by RRF to a broad candidate pool (top-100 to top-200), then a **cross-encoder reranker** on that shortlist to pick the final ~5-10 chunks. A SemEval-2026 hybrid RAG system used exactly this: RRF (k=10) → top-200 → bge-reranker-v2-m3 rerank of top-100 → optional α-weighted rescoring.
- **Reranker choice (self-host, permissive license): bge-reranker-v2-m3 (Apache-2.0), which is also multilingual** — a strong default. ColBERT-style late interaction (MaxSim) offers near-cross-encoder quality at much lower latency (ColBERT p50 ~23ms vs a cross-encoder's p99.9 >21s at 40 QPS) and is best when tail latency matters; it shines more as a first-stage retriever than a pure reranker. **License caveat: Jina Reranker v2 multilingual weights are CC-BY-NC-4.0 (non-commercial)** — avoid for self-hosted commercial use; Cohere Rerank is a managed API (not self-host).

### When multi-hop agentic retrieval is worth it (benchmark evidence)
- **Adaptive-RAG (Jeong, Baek, Cho, Hwang & Park, NAACL 2024, pp. 7036–7050, DOI 10.18653/v1/2024.naacl-long.389)** is the canonical pattern you want: a T5-Large classifier pre-assesses query complexity and routes among (a) no retrieval, (b) single-step retrieval, and (c) multi-step/iterative retrieval, and "can match always-expensive baselines with substantially lower cost." Lightweight routers are viable: the RAGRouter-Bench baseline study (arXiv:2604.03455) reports lexical TF-IDF features "outperform semantic sentence embeddings by 3.1 macro-F1 points," with a best classifier around ~92% accuracy / ~0.92 macro-F1 and roughly a quarter of tokens saved versus always using the most expensive path (verify exact figures against that paper's Table 2 before quoting).
- **Multi-hop is a categorical capability gap on compositional benchmarks, not a marginal one.** On MuSiQue (2-4 hop questions), NaiveRAG scores ~8.2 EM / 47 Recall@10 while iterative/decomposition methods reach ~39-44 EM and 75-90 Recall@10; the gap on HotpotQA and 2WikiMultiHopQA is smaller because those are often answerable with fewer hops. But the cost is real: structured iterative methods run ~2-3× slower than single-pass (e.g., ~8.1s vs ~3.3s on HotpotQA).
- **Graph/agentic methods do NOT help on single-fact lookups** — WildGraphBench (2026) found "flat baselines like NaiveRAG remain competitive on single-fact retrieval," and GraphRAG "can be more expensive than NaiveRAG or BM25 without clear gains for single-fact lookup." A robustness caveat: on noisy/ASR-corrupted input, multi-hop extensions can *amplify* errors (F1 gap 36-67% larger under entity-graph linking + iterative reformulation).
- **Conclusion:** route by complexity. Single-pass hybrid+rerank for the majority of queries; escalate to a bounded agentic loop (query decomposition + iterative retrieval + self-critique) only for multi-entity/multi-hop questions. Cap the loop iterations (bounded self-correction) to control latency and the error-amplification risk.

### Agentic patterns to layer in (in order of ROI)
1. **Query routing / complexity classification** (Adaptive-RAG) — biggest single win, cheap.
2. **Corrective RAG (CRAG, Yan et al., 2024, arXiv:2401.15884)** — a post-retrieval relevance grader that triggers query rewrite or web-search fallback when retrieved context is weak.
3. **Self-RAG (Asai et al., 2024)** — reflection tokens for retrieve-or-not and groundedness self-critique; a 2025 MDPI Electronics study of 12 RAG variants reported Self-RAG had the lowest hallucination rate (5.8%) among them.
4. **Iterative/multi-hop loops** (ReAct, IRCoT, Self-Ask) — only on the complex branch.
Modern surveys (Singh et al., arXiv:2501.09136; the System-1/System-2 reasoning survey arXiv:2506.10408) taxonomize these as reflection, planning, tool use, and multi-agent collaboration.

### GraphRAG approaches and Neo4j
- **Microsoft GraphRAG** (MIT): offline hierarchical community detection + community summaries; best for *global/cross-document aggregation* ("what themes span this corpus"), expensive to index.
- **LightRAG** (MIT): dual-level (low-level entity/relation exact-match + high-level thematic) retrieval with incremental updates; cheaper and faster than MS GraphRAG, better for entity-relationship queries.
- **HippoRAG 2** (Gutiérrez et al., ICML 2025, "From RAG to Memory," arXiv:2502.14802): dual-node KG (phrase + passage nodes) + Personalized PageRank + LLM triple filtering, "achieving a 7% improvement in associative memory tasks over the state-of-the-art embedding model while also exhibiting superior factual knowledge and sense-making memory capabilities" — i.e., it improves multi-hop/associative retrieval *without* degrading single-fact performance, and is cheaper to index than MS GraphRAG/RAPTOR/LightRAG. Notably its reference implementation pairs a vector store (pgvector) with Neo4j.
- **Neo4j's own stack**: `neo4j-graphrag-python` (first-party, Apache-2.0, long-term support) with `SimpleKGPipeline` (document parse → chunk → embed → schema-guided entity/relation extraction → entity resolution → write), plus the LLM Knowledge Graph Builder app. It natively supports Vector, Hybrid, GraphRAG, and Text2Cypher retrievers, and offers a native Qdrant retriever.
- **Verdict for you**: adopt the *simple document graph* from the reference paper first (proven, cheap, secure via curated Cypher tools). Add HippoRAG-2-style entity/PPR retrieval as the "GraphRAG extension point" if multi-hop associative queries become important. Treat MS GraphRAG as an optional offline "global summary report" generator, not the core retrieval path.

### Text2Cypher: use with care
LLM-generated Cypher is "the most flexible, but also the most unreliable" pattern. Neo4j released the Text2Cypher-2025v1 dataset (~44k instances) and fine-tuned models (e.g., Gemma-3-27B on HF). Fine-tuned open models get ~40% improvement on Text2Cypher tests; one GRPO-refined small model hit 85% execution accuracy on unseen schemas. But a 2026 study found the Neo4j Gemma-3-27B model unsatisfactory on complex schemas (it "consistently selected incorrect relationship and node labels"). **Recommendation: prefer curated parameterized Cypher tools (as the reference paper does) for the core; if you enable text2cypher, add schema filtering, regex/CyVer validation, and a ReAct correction loop.**

### Vector store selection (self-host, permissive license)
- **Qdrant (Apache-2.0, Rust)** — recommended default: sub-10ms p50 vector queries, strong filtered search, native sparse (SPLADE/miniCOIL) and ColBERT multi-vector support, native RRF (since v1.10), first-class Neo4j GraphRAG integration. Runs cleanly on AKS.
- **Milvus (Apache-2.0)** — for billion-scale; higher ops complexity.
- **Weaviate (BSD-3)** — built-in hybrid search + vectorizers; switched default fusion to Relative Score Fusion in v1.24.
- **pgvector (PostgreSQL license, permissive)** — best if you already run Postgres and are under ~50-100M vectors; HNSW (since 0.5.0) closed much of the performance gap.
- **Elasticsearch/OpenSearch for BM25**: **OpenSearch (Apache-2.0) is the right pick** — it added native RRF in 2.19 (Feb 2025). Note Elasticsearch's native RRF requires an Enterprise license, and Elastic's license is SSPL/Elastic-License (not OSI-permissive).
- **Neo4j built-in vector index vs separate store**: Neo4j vector search is an add-on to a graph engine; benchmarks show dedicated stores deliver ~10× lower vector latency. The proven production pattern (Lettria's GraphRAG at 100M+ embeddings, sub-200ms, reported +20-25% accuracy) keeps **Qdrant for vectors + Neo4j for relationships, joined by shared IDs**, with the Neo4j commit as the consistency gate. Use Neo4j's built-in vector index only for small corpora or to avoid a second system in an MVP.

### Entity Recognition (pluggable)
- **LLM-based extraction** (as in neo4j-graphrag SimpleKGPipeline, HippoRAG, LightRAG): most flexible, schema-guided, but slow and GPU-expensive; error propagation between NER and RE stages.
- **GLiNER / GLiNER2 (Apache-2.0)**: encoder-based zero-shot NER with natural-language labels; runs on CPU, "orders of magnitude faster than autoregressive LLM-based extraction." GLiNER2 unifies NER + classification + relation extraction with schema-driven JSON — well-suited to KG construction. Trade-off: on a 30-task query-parsing benchmark GLiNER got 53% fully-correct vs GPT-4.1-mini's 100%, but 15× faster (0.08s vs 1.21s).
- **spaCy (MIT)**: fast, mature, fixed entity types; good for high-volume classical NER.
- **Entity resolution/dedup**: neo4j-graphrag ships SpaCySemanticMatchResolver and FuzzyMatchResolver (RapidFuzz); HippoRAG uses synonym detection on phrase nodes.
- **Recommendation**: make NER an interface with two implementations — a fast GLiNER2/spaCy path for high-volume ingestion and an LLM-extraction path for high-value documents — plus a resolver stage.

### Hungarian / multilingual support (flag)
- **Embeddings**: **BGE-M3 (MIT) and multilingual-e5-large-instruct (MIT)** both explicitly list Hungarian ("hu") in their training languages. MMTEB (Enevoldsen et al., ICLR 2025, arXiv:2502.13595v4) found that "the best-performing publicly available model is multilingual-e5-large-instruct with only 560 million parameters," outperforming billion-parameter LLMs (e5-mistral-7b-instruct, GritLM-7B) especially on mid-to-low-resource languages — making it and BGE-M3 the strongest openly available choices for Hungarian. Caveat: an independent 2024 benchmark noted BGE-M3 shows "significant variations in performance" on Hungarian, and both models warn low-resource languages may degrade. **jina-embeddings-v3 supports Hungarian only in its broader 89-language set, NOT its top-30 "best performance" tier** — so prefer BGE-M3/e5 for Hungarian. Snowflake arctic-embed v2.0 lists Hungarian (74 langs) but only benchmarks DE/EN/ES/FR/IT.
- **NER**: **huBERT (SZTAKI-HLT) is SOTA for Hungarian** (set records on the Szeged NER corpus; a huBERT-based NerKor tagger reaches ~0.82 F1 on the ~30-type NerKor+Cars-OntoNotes++ scheme). **HuSpaCy** `hu_core_news_lg` reports NER F1 ≈ 0.867 (CPU-friendly), and transformer variants (huBERT / XLM-RoBERTa-large) score higher. GLiNER multilingual variants do NOT list Hungarian and have no published Hungarian benchmark — treat GLiNER as zero-shot/unvalidated for Hungarian and prefer huBERT/HuSpaCy.
- **Reranker**: bge-reranker-v2-m3 is multilingual and covers Hungarian.

### Web search as a pluggable tool
- **Tavily** is the de-facto default for LLM/RAG workflows (clean, chunked, source-attributed results; native LangChain/LangGraph integration; free Researcher tier ~1,000 credits/month) — but ~$8/1k pay-as-you-go can be costly at scale, and advanced tiers add latency. (One source reports Tavily was acquired by Nebius in Feb 2026.)
- **Brave Search API** — independent index, privacy-focused, ~$5/1k, strong agent-benchmark scores and low latency (~669ms).
- **SearXNG (AGPL-3.0)** — fully self-hostable metasearch; best fit for your self-host preference, at the cost of ops overhead. **License note: AGPL-3.0 is network-copyleft — keep it as a separate service behind your tool interface to avoid license entanglement with your code.**
- Integrate web search as a **CRAG-style fallback** (triggered when the relevance grader finds internal context insufficient) with explicit source attribution/citations. Keep the agent's tool count small (3-5); too many similar tools degrade selection accuracy.

### Chunking & indexing for Markdown
- **Header-based (structure-aware) splitting first, then recursive character splitting within sections** is the recommended default for Markdown/HTML — start at ~512 tokens with 10-20% overlap. The Vectara NAACL 2025 study (arXiv:2410.13070) found chunking configuration influences retrieval quality as much as the embedding-model choice, and that **semantic chunking's extra cost is often NOT justified** — fixed/recursive chunks frequently match or beat it. Chroma's evaluation found up to a 9% recall gap between best/worst strategies.
- Preserve document structure as metadata (header hierarchy, source lineage/position) — this doubles as your Neo4j document-graph structure (Document→Section→Chunk).
- Advanced options to evaluate later: Late Chunking (Günther et al., 2024) and Anthropic's Contextual Retrieval (prepend doc-level context to each chunk before embedding).

### Orchestration framework
- **LangGraph (MIT)** — recommended for the agentic control plane: explicit stateful graph, clear decision/branch points, bounded loops, LangSmith observability. Best fit for adaptive routing + bounded self-correction loops. (Since LangChain 1.0, Oct 2025, LangGraph is the execution engine LangChain agents run on.)
- **LlamaIndex (MIT)** — strongest for ingestion/indexing and retrieval; many production teams pair LlamaIndex (retrieval) + LangGraph (orchestration).
- **Haystack (deepset, Apache-2.0)** — clean typed components, production/enterprise focus (RBAC, monitoring); natively supports web-search fallback pipelines.
- **Custom** — viable given your team's experience, but LangGraph's state model saves boilerplate for the routing/loop logic.

### Evaluation strategy
- **RAGAS** — reference-free faithfulness/answer-relevancy/context-precision/context-recall; best for fast iteration. **DeepEval** — pytest-style, CI/CD-native, broadest metric library including agentic metrics; best for guarding a pipeline. **TruLens** — production monitoring/tracing (OpenTelemetry). **ARES** (Saad-Falcon et al., NAACL 2024) — automated eval with fine-tuned judges.
- Add **retrieval metrics** (nDCG@k, Recall@k, MRR) measured separately from generation metrics, and **always track latency and per-query cost** alongside quality — a pipeline that scores higher on faithfulness but takes 8s and 3 extra LLM calls may be worse in production. Note independent work (GroUSE, arXiv:2409.06595) shows LLM-judge faithfulness scorers have blind spots — don't treat a single automated score as ground truth.
- **Eval-driven loop**: build a golden query set from real user queries early; start with RAGAS for a baseline; add DeepEval tests in CI so every retriever/prompt change is gated; add TruLens once real traffic flows.

## Details

### Recommended reference architecture (simple-first, with marked extension points)

**Ingestion / Indexing pipeline (decoupled, batch or event-driven):**
```
Source docs ──▶ [Loader/Parser interface] ──▶ [Chunker: markdown-header → recursive, 512 tok/15% overlap]
     │                                              │
     │   (FUTURE: swap in your PDF→Markdown/OCR)     ├──▶ [Embedder interface: BGE-M3 / e5 via vLLM] ──▶ Qdrant (dense + sparse)
     │                                              ├──▶ [BM25 indexer] ──▶ OpenSearch (or Qdrant sparse)
     │                                              └──▶ [Document-graph writer] ──▶ Neo4j (Document→Section→Chunk)
     │
     └── EXTENSION POINT A (NER→KG): [NER interface: GLiNER2/spaCy | LLM extraction] ──▶ [Entity resolver] ──▶ Neo4j entity graph
```
Incremental updates: content-hash each chunk; upsert changed chunks into Qdrant + Neo4j in one transaction with the Neo4j commit as the consistency gate; support tombstones for deletes.

**Query / Retrieval pipeline:**
```
Query ──▶ [Router: complexity classifier]
              │
   simple ────┴──▶ FAST PATH: hybrid retrieve (BM25 ∪ dense) ─RRF─▶ top-100 ─▶ bge-reranker-v2-m3 ─▶ top-8 ─▶ LLM answer + citations
              │
  complex ────▶ AGENTIC PATH (LangGraph, bounded N iterations):
                  decompose → for each sub-q: hybrid retrieve + rerank (+ EXTENSION B: graph/entity-PPR retrieval via Neo4j Cypher tools)
                  → CRAG relevance grade → if weak: rewrite OR EXTENSION C: web search (Tavily/SearXNG) with attribution
                  → aggregate → self-critique/groundedness check → answer + citations
```
Extension points are feature-flagged interfaces so the fast path ships first and the agentic path/graph/NER/web-search are added without refactoring.

### Concrete Neo4j schema (document/lexical graph + optional entity graph)

Constraints & indexes (Neo4j 5.x):
```cypher
// Uniqueness constraints
CREATE CONSTRAINT doc_id   IF NOT EXISTS FOR (d:Document)  REQUIRE d.id   IS UNIQUE;
CREATE CONSTRAINT sec_id   IF NOT EXISTS FOR (s:Section)   REQUIRE s.id   IS UNIQUE;
CREATE CONSTRAINT chunk_id IF NOT EXISTS FOR (c:Chunk)     REQUIRE c.id   IS UNIQUE;
CREATE CONSTRAINT ent_id   IF NOT EXISTS FOR (e:Entity)    REQUIRE e.id   IS UNIQUE;

// Full-text (BM25-style) index for lexical search inside Neo4j
CREATE FULLTEXT INDEX chunkText IF NOT EXISTS FOR (c:Chunk) ON EACH [c.text];

// Native vector index (use for small corpora / MVP; else keep vectors in Qdrant)
CREATE VECTOR INDEX chunkEmbedding IF NOT EXISTS
FOR (c:Chunk) ON (c.embedding)
OPTIONS { indexConfig: { `vector.dimensions`: 1024, `vector.similarity_function`: 'cosine' } };
```

Node/relationship model:
```
(:Document {id, title, source_uri, lang, created_at})
  -[:HAS_SECTION]->(:Section {id, title, order})
  -[:HAS_CHUNK]->(:Chunk {id, text, embedding, token_count, order, content_hash})

(:Chunk)-[:NEXT]->(:Chunk)                    // sequential order for context expansion
(:Chunk)-[:LINKS_TO]->(:Document|:Section)     // cross-references (the reference paper's key edge)

// EXTENSION (entity graph):
(:Chunk)-[:MENTIONS]->(:Entity {id, name, type, aliases})
(:Entity)-[:RELATED_TO {type, evidence_chunk_id}]->(:Entity)
// Optional GraphRAG community layer:
(:Entity)-[:IN_COMMUNITY]->(:Community {id, summary, level})
```
BGE-M3 produces 1024-dim vectors (set `vector.dimensions` accordingly; multilingual-e5-large is also 1024). Keep `content_hash` for incremental re-indexing. Prefer **parameterized Cypher tools** exposed to the agent (e.g., `get_neighbors(chunk_id)`, `expand_section(section_id)`, `entities_between(a,b)`) over free-form text2cypher.

### Trade-off tables

**Fusion methods**
| Method | Needs score calibration | Tuning | When to use |
|---|---|---|---|
| RRF | No (rank-based) | k only (~60) | Default; heterogeneous BM25+dense |
| Weighted score fusion | Yes (fragile) | per-retriever weights | Only when scores are comparable/calibrated |
| Relative Score Fusion | Yes (normalized) | — | Weaviate default; when you need score magnitudes |
| Learned reranker (cross-encoder) | N/A (second stage) | model choice | Always, as stage-2 on the fused shortlist |

**Rerankers**
| Model | License | Latency | Notes |
|---|---|---|---|
| bge-reranker-v2-m3 | Apache-2.0 | medium | Recommended self-host default; multilingual (incl. Hungarian) |
| ColBERTv2 (late interaction) | Apache-2.0 | very low (p50 ~23ms) | Best for tight latency / high QPS; great first-stage too |
| Jina Reranker v2 multilingual | CC-BY-NC-4.0 | low | AVOID for self-host commercial (non-commercial weights) |
| Cohere Rerank | Managed API | n/a | Not self-hostable |

**Vector stores**
| Store | License | Strengths | Watch-outs |
|---|---|---|---|
| Qdrant | Apache-2.0 | sub-10ms, filtered search, sparse+ColBERT, native RRF, Neo4j integration | — (recommended) |
| Milvus | Apache-2.0 | billion-scale | ops complexity |
| Weaviate | BSD-3 | built-in hybrid | fusion default changed across versions |
| pgvector | PostgreSQL (permissive) | reuse Postgres | ~50-100M vector ceiling |
| OpenSearch | Apache-2.0 | BM25 + native RRF (2.19) | heavier than Qdrant for pure vectors |
| Elasticsearch | SSPL/Elastic (non-OSI) | mature | native RRF needs Enterprise license |
| Neo4j vector index | GPLv3 core / commercial | co-located with graph | ~10× higher vector latency; MVP only |

**GraphRAG frameworks**
| Framework | License | Best for | Cost |
|---|---|---|---|
| Simple document graph (ref paper) | your code | structure-aware retrieval, security | very low |
| HippoRAG 2 | Apache-2.0 (repo) | associative/multi-hop without hurting single-fact | low index cost |
| LightRAG | MIT | entity-relationship queries, incremental updates | low |
| Microsoft GraphRAG | MIT | global/cross-doc summarization | high index cost |
| neo4j-graphrag-python | Apache-2.0 | first-party KG build + retrievers (Vector/Hybrid/GraphRAG/Text2Cypher) | medium |

**NER options**
| Option | License | Speed | Hungarian |
|---|---|---|---|
| GLiNER2 | Apache-2.0 | fast (CPU) | not listed / unvalidated |
| spaCy | MIT | fast | HuSpaCy hu_core_news (F1≈0.867) |
| huBERT | permissive | medium (GPU) | SOTA Hungarian (~0.82 F1 on NerKor) |
| LLM extraction | model-dependent | slow | strong (multilingual LLM) |

**Orchestration**
| Framework | License | Strength | Note |
|---|---|---|---|
| LangGraph | MIT | stateful graphs, bounded loops, routing | recommended control plane |
| LlamaIndex | MIT | ingestion/indexing/retrieval | pair with LangGraph |
| Haystack | Apache-2.0 | typed components, enterprise, web fallback | strong alternative |
| Custom | — | full control | more boilerplate |

## Recommendations

**Stage 0 — Fast-path MVP (weeks 1-4).** Ship single-pass hybrid RAG: markdown header→recursive chunking (512 tok/15% overlap), BGE-M3 embeddings served via vLLM on AKS, Qdrant (dense+sparse) + OpenSearch or Qdrant-sparse for BM25, RRF fusion, bge-reranker-v2-m3 rerank to top-8, LLM answer with inline citations. Stand up Neo4j with the Document→Section→Chunk graph in parallel (cheap, enables structure-aware retrieval later). Build a golden query set and wire RAGAS from day one. **Benchmark to beat:** faithfulness and context-recall on your golden set; measure p50/p95 latency and per-query cost.

**Stage 1 — Adaptive routing + corrective loop (weeks 5-8).** Add the LangGraph router (start with a keyword/heuristic or small classifier — Adaptive-RAG shows a light classifier suffices) and a CRAG relevance grader with query rewrite. **Threshold to escalate to Stage 2:** if >20-30% of queries return low-faithfulness answers, or your query logs show multi-entity/multi-hop questions, build the agentic branch.

**Stage 2 — Agentic multi-hop branch (weeks 9-12).** Add query decomposition + bounded (≤3-4 iteration) iterative retrieval on the complex branch only, with self-critique. Add graph retrieval via **parameterized Neo4j Cypher tools** (not text2cypher). **Threshold:** adopt only if it improves EM/F1 on your multi-hop eval slice by a margin that justifies the ~2-3× latency; if not, keep it gated to a narrow query class.

**Stage 3 — Optional extensions (as needed).** (a) NER→entity graph (GLiNER2 for volume, LLM extraction for high-value docs; huBERT/HuSpaCy for Hungarian) and HippoRAG-2-style PPR retrieval if associative queries matter. (b) Web-search fallback (SearXNG self-hosted if you want full control; Tavily for speed) triggered by the CRAG grader, with source attribution. (c) MS GraphRAG offline for periodic global-summary reports.

**Cross-cutting:** keep every component behind a clean interface (loader, chunker, embedder, retriever, reranker, NER, graph store, web search) so the future PDF→Markdown/OCR pipeline and any model swap are drop-in. Re-verify reranker/embedding choices quarterly against your own eval set — leaderboards shift and your domain may reorder them.

## Caveats
- **Vendor/marketing sources**: several comparison points (vector-store latency claims, Lettria's +20-25% accuracy, framework "retrieval accuracy" percentages) come from vendor blogs or practitioner posts, not peer-reviewed benchmarks — treat as directional. Prefer your own eval numbers before committing.
- **Benchmark transfer**: HotpotQA/MuSiQue/2Wiki results are on Wikipedia-style open-domain QA; your document-intelligence corpus may behave differently. The reference paper itself is on MoNaCo/Wikipedia and is a v2 preprint (Jul 2026) — check for a peer-reviewed final version and its released evaluation KG.
- **GraphRAG evaluation is contested**: a 2025 study found reported GraphRAG gains partly stem from LLM-judge position/length/verbosity biases that shrink or vanish when corrected — validate graph gains with unbiased protocols.
- **Router figures**: the "0.928 macro-F1 / 93.2% / ~28% token savings" numbers for a lightweight router come from the RAGRouter-Bench baseline paper (arXiv:2604.03455); confirm the exact values against its Table 2 before quoting, as reported best figures cluster around ~0.92 macro-F1.
- **Hungarian**: embedding quality on Hungarian is weaker than on high-resource languages even for the best models; validate on a Hungarian eval slice, and prefer huBERT/HuSpaCy over GLiNER for Hungarian NER.
- **Licenses to watch**: SearXNG is AGPL-3.0 (network copyleft — isolate as a service); Neo4j Community is GPLv3 (Enterprise features are commercial); Elasticsearch is SSPL (non-OSI); Jina reranker v2 multilingual weights are CC-BY-NC (non-commercial). Qdrant, Milvus, OpenSearch, LangGraph, LlamaIndex, Haystack, GLiNER2, bge-reranker, BGE-M3 and multilingual-e5 are all permissively licensed (Apache/MIT/BSD/PostgreSQL).
- **Dates**: current as of August 2026; the RAG tooling landscape moves fast (e.g., LangChain 1.0 made LangGraph the execution engine in Oct 2025; Tavily's reported Nebius acquisition Feb 2026) — re-verify before procurement.