---
aliases: ["LMCache"]
tags: [rag, serving, kv-cache, infrastructure, tool]
sources: [The Everything Else Map of SOTA RAG Optimization (2024–2026) - A Decision-Ready Survey for a Greenfield Agentic RAG System.md, Deep-Dive - 10 RAG Optimization Techniques for a Self-Hosted Agentic Stack.md]
created: 2026-08-27
updated: 2026-08-27
---

# LMCache

The production KV-cache layer for [[vLLM]], and the home of **CacheBlend**. The biggest RAG-specific serving win available. See [[KV-Cache Reuse for RAG]].

## CacheBlend (EuroSys 2025 Best Paper, arXiv 2405.16444)

The problem: RAG concatenates retrieved chunks that vary per query and sit at varying positions, so standard prefix caching only helps the first chunk and prefill is recomputed every query.

The mechanism: pre-compute each chunk's KV cache independently, then at serving time (1) reuse all chunks' KV **regardless of position**, and (2) selectively recompute the KV of a small subset of *cross-attention-deviant* tokens to restore full-prefill quality.

Reported (Yao et al., UChicago/Microsoft): *"minimal loss in quality compared with full KV recompute, with 5%–18% selective recompute ratio"* and *"reduces TTFT by 2.2~3.3× and increases throughput by 2.8~5× under negligible quality drop."* The small recompute pipelines with KV retrieval, so slower/cheaper storage tiers can hold the cache without adding latency.

## LMCache itself

A standalone daemon — **no fate-sharing with the engine, so the cache survives engine crashes**. Multi-tier: GPU → CPU DRAM (hot) → local disk → remote shared backend. Integrates into vLLM v1 via the KV-connector API (`--kv-transfer-config` with `kv_connector='LMCacheConnectorV1'`); default chunking 256 tokens; maintains a token-sequence→KV index enabling **cross-request and cross-instance** hits.

Reported: 3–10× latency reduction versus recompute on the CPU tier; the LMCache paper (arXiv 2510.09665) reports *"1.9 to 8.1× smaller TTFT"* at QPS=1 and *"2.3–14× higher query processing rate…than the strongest baseline across five evaluated models."* A concrete long-context case (VAST Data on DGX SuperPOD): TTFT at 128K context fell from over 11 seconds to 1.5.

**Production readiness:** LMCache "graduated to production in January 2026"; used by Google Cloud GKE Inference, CoreWeave and Cohere. The vLLM Production Stack integration ships in LMCache 0.4+.

## AKS integration

Deploy via the vLLM Production Stack Helm chart: `lmcacheConfig.enabled: true`, set `cpuOffloadingBufferSize`, use a PVC for the disk tier. **For multi-replica, run a remote cache server** so requests routed to different pods share KV. Pair with KV-aware routing.

## Adoption order

1. vLLM `--enable-prefix-caching` — free win for shared system prompts.
2. LMCache CPU offload.
3. CacheBlend blending — **only after measuring answer-quality parity**, since non-prefix reuse trades a small quality delta for speed and chunk ordering affects results (see CacheWeaver). Keep chunk boundaries stable.

Alternatives: RAGCache (arXiv 2404.12457), TurboRAG (2410.07590), EPIC/position-independent caching (2410.15332), CacheCraft, CacheClip (2510.10129).

## Related

[[KV-Cache Reuse for RAG]] · [[vLLM]] · [[Chunking Strategies]]
