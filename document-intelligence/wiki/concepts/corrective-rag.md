---
aliases: ["Corrective RAG"]
tags: [rag, agentic, reliability, architecture]
sources: [SOTA Agentic RAG Reference Architecture - A System-Design Session Brief.md, Query Modification for a SOTA Agentic RAG System - Taxonomy Benchmarks and a Staged Design.md]
created: 2026-08-27
updated: 2026-08-27
---

# Corrective RAG

**CRAG** (Yan et al., 2024, arXiv 2401.15884): a post-retrieval **relevance grader** that inspects the retrieved context before generation and takes corrective action when it is weak.

Second-highest ROI agentic pattern after [[Adaptive RAG Routing]], and cheap.

## The three-way branch

| Grade | Action |
|---|---|
| **Correct** | refine and generate |
| **Ambiguous** | merge knowledge-base results with web results |
| **Incorrect** | rewrite the query and fall back to web search, with explicit source attribution |

## Why the grader is the right gate for expensive tricks

CRAG's real architectural value is that it turns fragile techniques into **conditional** ones. Query expansion (HyDE, Query2Doc) *hurts* when the retriever is strong, the domain is entity-heavy, or the LLM lacks corpus knowledge — so it must never be always-on. Gating it behind the grader means it fires only when relevance is genuinely low **and** the query looks unfamiliar or vocabulary-mismatched. See [[Query Rewriting and Expansion]].

The same logic gates the web-search fallback: it is a **CRAG-triggered** tool, not a routine one.

## Web search as a pluggable tool

| Option | Notes |
|---|---|
| **Tavily** | de-facto default for RAG workflows (clean chunked source-attributed results, native LangChain/LangGraph integration, free tier ~1,000 credits/month) — but ~$8/1k pay-as-you-go is costly at scale, and advanced tiers add latency. Reportedly acquired by Nebius in Feb 2026 |
| **Brave Search API** | independent index, privacy-focused, ~$5/1k, strong agent-benchmark scores, low latency (~669 ms) |
| **SearXNG** | fully self-hostable metasearch — best fit for a self-host preference, at the cost of ops overhead. **AGPL-3.0 network copyleft: keep it a separate service behind your tool interface** |

Keep the agent's tool count small (3–5); too many similar tools degrade selection accuracy.

## The related reflection patterns

- **Self-RAG** (Asai et al., 2024) — reflection tokens for retrieve-or-not and groundedness self-critique. A 2025 MDPI *Electronics* study of 12 RAG variants reported Self-RAG had the **lowest hallucination rate (5.8%)** among them.
- Modern surveys (Singh et al., arXiv 2501.09136; the System-1/System-2 survey arXiv 2506.10408) taxonomize these as reflection, planning, tool use and multi-agent collaboration.

CRAG grades *retrieval*; a groundedness gate checks the *answer*. Both belong — see [[Hallucination Detection in RAG]].

## Related

[[Adaptive RAG Routing]] · [[Query Rewriting and Expansion]] · [[Hallucination Detection in RAG]] · [[LangGraph]]
