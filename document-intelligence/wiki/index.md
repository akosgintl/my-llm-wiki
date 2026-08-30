# Index

Master catalog of all wiki pages. Updated on every ingest.

## Sources

- [[Open-Source OCR VDU Models for Self-Hosted AKS Pipelines]] — the model-landscape survey everything else builds on
- [[Serving Recipes for OCR VDU on vLLM SGLang and RunPod]] — per-model vLLM/SGLang configs and RunPod API v2 templates
- [[API Call Strategy Images Concurrency and Parallelism]] — request shapes, concurrency sizing, the parallelism stack, load balancing
- [[Rasterization Encoding and the Real Bottlenecks]] — critical pass on resolution, encoding, base64 and gateway language
- [[Language Models Confidence and the Deployment Map]] — the pivot: Hungarian, Rule 0/0.5, sovereignty, fine-tuning, confidence
- [[Self-Hosted OCR VDU on AKS The Complete Study]] — consolidated master document merging sources 1–5 plus new material
- [[arXiv and GitHub Technical Review of Missed Details]] — per-model pass over papers and repos for under-specified facts
- [[Deep Technical Review of Missed Details and Pipeline Implications]] — deeper second pass with exact numbers and repo verdicts
- [[Local PDF to Markdown and Document Parsing State of the Art]] — the wider local-parsing ecosystem, frameworks and licensing
- [[Tiered PDF Pipeline Architecture]] — buildable three-tier, permissive-only, Hungarian-first reference design
- [[SOTA Agentic RAG Reference Architecture]] — simple-first hybrid RAG core with an adaptive router and a document graph
- [[Query Modification for Agentic RAG]] — taxonomy of query transformations, what helps and what quietly hurts
- [[The Everything Else Map of SOTA RAG Optimization]] — ten-stage breadth sweep with adopt/extend/skip verdicts
- [[Deep-Dive 10 RAG Optimization Techniques]] — implementation-ready configs, licenses and dependencies for the top ten

## Entities

### OCR and document-parsing models

- [[GLM-OCR]] — 0.9B MIT two-stage VLM; fastest in class, but 8 languages and no Hungarian
- [[PaddleOCR-VL]] — 0.9B Apache-2.0, Hungarian confirmed, best on messy multilingual scans, not fine-tunable
- [[DeepSeek-OCR]] — optical compression, 200k pages/day, repetition loops and an AGPL dependency; **OCR-2 is the fine-tune base**, still weak multi-script
- [[Baidu Unlimited-OCR]] — R-SWA constant-KV attention; dozens of pages in one request, and no language roster at all
- [[dots.ocr]] — 1.7B MIT; the independent-benchmark champion, slow and VRAM-hungry
- [[MinerU]] — full ingestion platform; two stages inside one 1.2B model, collapses on non-Latin
- [[HunyuanOCR]] — the faithfulness outlier (CHAOS-Bench 14.15 vs 3–6.3), NOASSERTION license
- [[olmOCR]] — fully open English linearization; RLVR-from-unit-tests is its transferable idea
- [[LightOnOCR]] — 1B European parser; card declares 11 languages, **Hungarian not among them**
- [[Granite-Docling]] — IBM's 258M DocTags model, the VLM path inside Docling
- [[Marker and Chandra]] — strong scores, blocked by OpenRAIL-M revenue caps
- [[Other OCR Contenders]] — Nanonets, Nemotron, Mistral OCR 4, PP-StructureV3, GOT-OCR2, Nougat
- [[OvisOCR2]] — 0.8B Apache-2.0 on a Qwen3.5 backbone; the highest OmniDocBench claim, and self-reported
- [[Qwen Model Family]] — the recognition-slot front-runner; 3.5 is natively multimodal at the right sizes, and the last generation that is

### Parsing tools and pipeline components

- [[Docling]] — MIT orchestration framework and DoclingDocument schema, now an LF project
- [[liteparse]] — Apache-2.0 Rust heuristic extractor behind LlamaParse; the Tier-1 fast path
- [[pymupdf4llm]] — fastest born-digital extractor, AGPL-3.0, fails silently on scans
- [[pypdfium2]] — the permissive rasterization workhorse; the tool/licence matrix lives here
- [[glmocr SDK]] — the pipeline kept after its model was rejected; env-var swap and the adapter shim
- [[PP-DocLayout-V3]] — Apache-2.0 detector that also emits reading order; the ONNX normalization gotcha
- [[PP-OCRv6]] — 34.5M CTC recognizer used as a hallucination cross-check
- [[Tesseract]] — handles ő/ű well on clean scans, and emits the confidence VLMs cannot

### Serving and infrastructure

- [[vLLM]] — the one serving contract everything speaks; universal OCR flags and the frontend bottleneck
- [[SGLang]] — secondary engine, mandatory for Unlimited-OCR's Hopper-only long-context path
- [[LMCache]] — CacheBlend KV-cache reuse; the biggest RAG-specific serving win
- [[LiteLLM]] — pragmatic load balancer and the multi-cloud seam
- [[RunPod]] — API v2 templates, serverless caveats, no MIG
- [[Unsloth]] — best OCR fine-tuning tooling; requires their modified DeepSeek checkpoint
- [[LLaMA-Factory]] — official GLM-OCR recipe, and the undocumented MTP-merge risk
- [[neural-maze production-ocr-course]] — closest public AKS reference; copy the patterns, not the numbers

### RAG stack

- [[Qdrant]] — Apache-2.0 default vector store; quantization, multi-vector and native RRF
- [[OpenSearch]] — the BM25 and learned-sparse leg; native RRF since 2.19
- [[Neo4j]] — lightweight document graph plus optional entity graph, vectors kept elsewhere
- [[LangGraph]] — MIT agentic control plane for routing and bounded loops
- [[LlamaIndex]] — ingestion and retrieval patterns; parent-child and auto-merging
- [[BGE-M3]] — MIT multilingual embedder with the only published Hungarian retrieval evidence
- [[Qwen3-Embedding]] — Apache-2.0 SOTA multilingual embedder; the proposed BGE-M3 replacement
- [[multilingual-e5-large-instruct]] — 560M MMTEB standout on mid-to-low-resource languages
- [[bge-reranker-v2-m3]] — Apache-2.0 multilingual cross-encoder; the stage-2 default
- [[ColPali]] — late-interaction retrieval over page images; an architecture, not a model, and it bypasses OCR entirely
- [[Qwen3-VL-Embedding]] — the single-vector multimodal alternative to ColPali; one vector per page instead of ~1,030
- [[LettuceDetect]] — MIT token-level groundedness detector with Hungarian coverage
- [[Provence]] — best context pruner, but CC-BY-NC weights — do not deploy commercially
- [[huBERT]] — SOTA Hungarian NER, and explicitly weak as a retrieval embedder
- [[HuSpaCy]] — CPU-friendly Hungarian NER for high-volume ingestion
- [[GLiNER]] — fast zero-shot NER, but Hungarian is unlisted and unvalidated

### Benchmarks

- [[OmniDocBench]] — the dominant benchmark, saturated and vendor-reported
- [[olmOCR-Bench]] — unit-test-style, English-only, with a header/footer scoring artifact
- [[OCR Arena]] — blind community ELO; where the vendor ranking inverts
- [[Multi-Script OCR Benchmarks]] — socOCRbench, GlotOCR and MORE reorder the field
- [[CHAOS-Bench]] — character-level faithfulness under visual-vs-prior conflict
- [[EXTRACTCONF]] — the 40-feature calibrated-confidence menu, and its 3-feature shortcut

### Organizations

- [[Baidu]] — PaddleOCR-VL, PP-DocLayout-V3, PP-OCRv6, Unlimited-OCR; both ends of the design space
- [[Zhipu AI]] — GLM-OCR and the SDK that outlived it
- [[DeepSeek AI]] — the optical-compression thesis, and two inherited liabilities
- [[OpenDataLab]] — MinerU and OmniDocBench; model and benchmark from one house
- [[IBM Research]] — Docling and Granite-Docling; the avoid-OCR-where-possible strategy
- [[Allen Institute for AI]] — olmOCR and olmOCR-Bench, fully open
- [[Tencent]] — HunyuanOCR and CHAOS-Bench; optimized for not inventing text
- [[Alibaba Qwen Team]] — touches the most slots on both the OCR and RAG side
- [[Anthropic]] — Contextual Retrieval and the long-context guidance, reimplemented self-hosted

## Concepts

### Methodology and evaluation

- [[Golden Set and Eval Harness]] — Rule 0: the most valuable artifact in the project
- [[Pipeline as Platform, Model as Config]] — Rule 0.5: the inversion that kills the version treadmill
- [[Benchmark Saturation]] — why the leaderboards produce wrong decisions
- [[Hungarian OCR and the Double Acute Test]] — the fact that re-ranked every model
- [[Permissive Licensing Constraints]] — what a permissive-only rule actually eliminates

### OCR failure modes and reliability

- [[Repetition Loops in VLM OCR]] — the #1 production failure, and four control patterns
- [[Linguistic Crutch and Faithfulness]] — when the model silently corrects what it can see clearly
- [[Confidence Engineering]] — confidence as a pipeline property, not a model feature
- [[The OCR-to-Text Boundary Limit]] — anything visual dies at the OCR-to-text handoff

### Pipeline and performance

- [[Rasterization at Model-Native Resolution]] — the single biggest optimization in the pipeline
- [[Base64 and Image Transport]] — the cargo cult, the four real costs, and the bottleneck audit
- [[Document Fan-Out and Fan-In]] — request shapes, the 200-page playbook, failure semantics
- [[vLLM Continuous Batching and Concurrency Sizing]] — the 1.25× in-flight rule
- [[Parallelism Stack for OCR Serving]] — six layers, and why the middle is a trap
- [[MIG and GPU Sharing]] — MIG > MPS > time-slicing, and where the banished T4s belong
- [[Load Balancing Inference Pools]] — why a plain Kubernetes Service is the wrong answer
- [[Layout Stage Economics]] — three placement tiers for a detector 10–50× cheaper than recognition

### Architecture and model internals

- [[Two-Stage vs End-to-End Document Parsing]] — the fork that decides request shape and ops surface
- [[Optical Compression]] — the accuracy-vs-tokens curve, and why it is also a hallucination setting
- [[R-SWA Reference Sliding Window Attention]] — constant KV cache, and why the encoder stays frozen
- [[Multi-Token Prediction and Speculative Decoding]] — the one place speculation pays, and its open risk
- [[Tiered Page Routing]] — route each page to the cheapest tier that can handle it

### Adaptation

- [[LoRA Fine-Tuning for OCR]] — freeze the encoder, LoRA the decoder, train format-native targets
- [[Born-Digital Self-Labeling]] — two cheats that make training data cost nothing
- [[EU Sovereign GPU Hosting]] — CLOUD Act follows incorporation, not geography

### RAG retrieval

- [[Hybrid Retrieval and RRF]] — rank-based fusion, and the mistake that sinks production hybrids
- [[Two-Stage Retrieve-Then-Rerank]] — the architecture that decides what else is worth doing
- [[Learned Sparse Retrieval]] — miniCOIL vs SPLADE, and the thin Hungarian coverage
- [[Embedding Quantization and MRL]] — ~80% storage cut at near-parity recall, with rescoring
- [[Chunking Strategies]] — as influential as the embedding model; parent-child by default
- [[Contextual Retrieval]] — the highest single-ROI ingestion technique
- [[RAPTOR Hierarchical Summarization]] — cluster-and-summarize trees, and their update problem
- [[GraphRAG and Document Graphs]] — a lightweight document graph beats a heavyweight ontology
- [[Text2Cypher]] — flexible, unreliable, and a security surface

### RAG orchestration and generation

- [[Adaptive RAG Routing]] — the biggest and cheapest agentic win
- [[Corrective RAG]] — the grader that turns fragile techniques into conditional ones
- [[Query Rewriting and Expansion]] — light rewriting helps, heavy expansion often hurts
- [[Query Decomposition and Multi-Hop]] — categorical gains where it applies, pure cost where it does not
- [[KV-Cache Reuse for RAG]] — the exact inverse of the OCR-side prefix-caching advice
- [[Context Pruning and Lost-in-the-Middle]] — cheap wins between reranking and generation

### RAG reliability

- [[Hallucination Detection in RAG]] — the inline groundedness gate, and what it is not a substitute for
- [[Indirect Prompt Injection]] — OWASP LLM01, ≥80% attack success at <0.1% poison rate
- [[RAG Evaluation]] — the downstream twin of the golden-set discipline

## Synthesis

- [[Hungarian Model Decision Matrix]] — all 10 recognition candidates on one table, plus the probe order
- [[The OCR-to-RAG Seam]] — the join the corpus never specifies: what must cross from OCR into retrieval
- [[Cost per Page Model]] — every throughput figure converted to dollars per 1,000 pages
- [[Open Inputs and Corpus Profile]] — everything still blocking a decision, sorted by who can answer it
- [[Coverage and Exclusion Register]] — every model this wiki names, with a dated verdict and the evidence behind it
- [[How to Query This Wiki]] — what the 120 pages can answer, and the five corpus facts they never can
