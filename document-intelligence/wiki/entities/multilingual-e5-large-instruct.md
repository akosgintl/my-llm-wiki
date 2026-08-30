---
aliases: ["multilingual-e5-large-instruct"]
tags: [rag, embeddings, multilingual, model]
sources: [SOTA Agentic RAG Reference Architecture - A System-Design Session Brief.md, The Everything Else Map of SOTA RAG Optimization (2024–2026) - A Decision-Ready Survey for a Greenfield Agentic RAG System.md]
created: 2026-08-27
updated: 2026-08-27
---

# multilingual-e5-large-instruct

MIT, 560M params, 1024 dims. The third viable multilingual embedder alongside [[BGE-M3]] and [[Qwen3-Embedding]], and the cheapest of the three.

Its standout evidence is from **MMTEB** (Enevoldsen et al., ICLR 2025, arXiv 2502.13595v4), which found that *"the best-performing publicly available model is multilingual-e5-large-instruct with only 560 million parameters"* — outperforming billion-parameter LLM embedders (e5-mistral-7b-instruct, GritLM-7B) **especially on mid-to-low-resource languages**. That is precisely the regime Hungarian sits in, and it explicitly lists `hu` among its training languages.

Later MMTEB aggregate scores put it at 63.22 Mean / 57.1 Retrieval — ahead of BGE-M3's 59.56/54.6 but behind Qwen3-Embedding.

**Position:** the small/cheap arm. Worth including in a Hungarian bake-off precisely because it punches above its parameter count on mid-resource languages, and because at 560M it is trivially servable. Like every candidate, its Hungarian number has to be measured rather than read off a leaderboard.

## Related

[[BGE-M3]] · [[Qwen3-Embedding]] · [[RAG Evaluation]]
