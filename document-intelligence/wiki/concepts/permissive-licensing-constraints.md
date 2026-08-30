---
aliases: ["Permissive Licensing Constraints"]
tags: [licensing, compliance, methodology]
sources: [Open-Source OCR per Visual Document Understanding Models for Self-Hosted AKS Pipelines - A Comparative Investigation (August 2026).md, ocr-rasterize-encoding-bottlenecks.md, Deep Technical Review - Open-Source OCR - Visual Document Understanding Models - Missed Details and Pipeline Implications.md, Local PDF-Markdown and Document Parsing - State of the Art (August 2026).md, tiered-pdf-pipeline-architecture.md, SOTA Agentic RAG Reference Architecture - A System-Design Session Brief.md, Deep-Dive - 10 RAG Optimization Techniques for a Self-Hosted Agentic Stack.md, The Everything Else Map of SOTA RAG Optimization (2024–2026) - A Decision-Ready Survey for a Greenfield Agentic RAG System.md]
created: 2026-08-27
updated: 2026-08-27
---

# Permissive Licensing Constraints

A permissive-only constraint (MIT / Apache-2.0 / BSD) is not a formality — it **eliminates specific best-in-class components** across both halves of the stack. Licensing is arguably the biggest practical differentiator in local document parsing.

## The blocked list

| Component | Problem | Replacement |
|---|---|---|
| **PyMuPDF / [[pymupdf4llm]]** | **AGPL-3.0** — and it is everyone's default rasterizer, so it propagates | [[pypdfium2]] or `pdftoppm` |
| **[[DeepSeek-OCR]] toolchain** | imports AGPL PyMuPDF; **Issue #223 disputes the MIT claim** ("becomes a derivative work") | replicate rasterization with pypdfium2, don't vendor their scripts |
| **[[Marker and Chandra]]** | modified **OpenRAIL-M** weights: free only under $2M revenue/funding and "cannot be used competitively with our API"; marker code GPL-3.0 | [[Docling]] (MIT) + [[PaddleOCR-VL]] (Apache-2.0) |
| Surya weights, DocLayout-YOLO | OpenRAIL-M / AGPL | [[PP-DocLayout-V3]] (Apache-2.0) |
| **[[HunyuanOCR]]** | **NOASSERTION** | counsel review before commercial use |
| [[MinerU]] | moved off AGPLv3 (Apr 2026) to a custom Apache-based license, but with attribution conditions and high-usage commercial thresholds | optional — add only if its hybrid mode wins |
| **[[Provence]]** | official weights **CC-BY-NC-4.0** (one README snapshot says cc-by-4.0; the model card and Naver's blog say NC) | train your own pruner, or RECOMP-style extraction |
| **[[ColPali]] backbones** | adapters MIT, but PaliGemma is Gemma-licensed and ColQwen2.5 checkpoints show **conflicting Apache-2.0 vs Qwen Research License tags** | pick a checkpoint whose backbone is genuinely Apache-2.0 |
| Jina Reranker v2 multilingual | CC-BY-NC-4.0 | [[bge-reranker-v2-m3]] |
| Elasticsearch | SSPL/Elastic (non-OSI); native RRF needs Enterprise | [[OpenSearch]] |
| [[Neo4j]] Community | GPLv3 core (Enterprise commercial) | accept, or isolate |
| SearXNG | AGPL-3.0 network copyleft | keep as a **separate service behind a tool interface** |
| Meta PromptGuard 2 | Llama license — not permissive | ProtectAI `deberta-v3-base-prompt-injection-v2` |
| NV-Embed-v2, EmbeddingGemma / gemini-embedding | non-commercial / CC-BY (Gemma) | [[Qwen3-Embedding]], [[BGE-M3]] |

## The clean list

MIT: [[Docling]], [[BGE-M3]], multilingual-e5-large, [[LettuceDetect]], [[LangGraph]], [[LlamaIndex]], [[GLM-OCR]] weights, [[dots.ocr]], [[Baidu Unlimited-OCR]].
Apache-2.0: [[PaddleOCR-VL]], [[PP-DocLayout-V3]], [[PP-OCRv6]], [[liteparse]], [[olmOCR]], [[Granite-Docling]], [[Qwen3-Embedding]], Qwen3-Reranker, [[bge-reranker-v2-m3]], [[Qdrant]], Milvus, [[OpenSearch]], Haystack, [[GLiNER]]2, Graphiti, neo4j-graphrag-python, [[Qwen Model Family]].
BSD/PostgreSQL: [[pypdfium2]], Weaviate, pgvector.

## Three rules

**1. AGPL propagates through defaults.** The PyMuPDF case is the pattern: a permissive project pulls an AGPL library and the claim is disputed. Standardizing on pypdfium2 or pdftoppm is **cheap now and expensive later**. (`mutool` via subprocess of an *unmodified* binary is generally considered safe across the process boundary — but get legal sign-off. Poppler's GPL-2 CLI is safe the same way.)

**2. Verify the weights license per checkpoint, not per family.** ColQwen2.5 community checkpoints carry contradictory tags for the *same* model family. Training-data licenses (`vidore/colpali_train_set` is CC-BY-NC-4.0) matter only if you retrain.

**3. Isolate network-copyleft services behind an interface.** SearXNG stays a separate service; the boundary is what prevents license entanglement with your code.

## Related

[[Tiered Page Routing]] · [[pypdfium2]] · [[Provence]] · [[ColPali]] · [[EU Sovereign GPU Hosting]]
