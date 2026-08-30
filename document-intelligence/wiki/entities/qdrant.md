---
aliases: ["Qdrant"]
tags: [rag, vector-store, retrieval, infrastructure]
sources: [SOTA Agentic RAG Reference Architecture - A System-Design Session Brief.md, The Everything Else Map of SOTA RAG Optimization (2024–2026) - A Decision-Ready Survey for a Greenfield Agentic RAG System.md, Deep-Dive - 10 RAG Optimization Techniques for a Self-Hosted Agentic Stack.md]
created: 2026-08-27
updated: 2026-08-27
---

# Qdrant

Apache-2.0, Rust. The recommended default vector store for a self-hosted agentic RAG stack.

## Why it wins the slot

Sub-10 ms p50 vector queries, strong filtered search, **native sparse** (SPLADE/miniCOIL) and **ColBERT multi-vector** support, native RRF since v1.10, and a first-class [[Neo4j]] GraphRAG integration. Runs cleanly on AKS.

| Alternative | License | When |
|---|---|---|
| Milvus | Apache-2.0 | billion-scale; higher ops complexity; native CAGRA + DiskANN |
| Weaviate | BSD-3 | built-in hybrid; fusion default changed across versions (Relative Score Fusion since 1.24) |
| pgvector | PostgreSQL | if you already run Postgres and are under ~50–100M vectors; HNSW since 0.5.0 |
| [[OpenSearch]] | Apache-2.0 | the BM25 side, not the vector side |
| Elasticsearch | SSPL/Elastic (non-OSI) | native RRF needs an Enterprise license — avoid |
| Neo4j vector index | GPLv3 core | ~10× higher vector latency; MVP or small corpora only |

The proven production pattern is **Qdrant for vectors + Neo4j for relationships, joined by shared IDs**, with the Neo4j commit as the consistency gate — not Neo4j's built-in vector index. See [[GraphRAG and Document Graphs]].

## Storage optimization

Qdrant is where [[Embedding Quantization and MRL]] actually gets configured:

```json
{
  "vectors": {"size": 1024, "distance": "Cosine"},
  "quantization_config": {"scalar": {"type": "int8", "quantile": 0.99, "always_ram": true}},
  "hnsw_config": {"on_disk": true}
}
```

Query with `params: {quantization: {rescore: true, oversampling: 2.0}}`. Scalar quantization does **not** rescore by default — enable it explicitly. Binary/TurboQuant methods do. Qdrant 1.15+ supports asymmetric query encoding (`query_encoding='scalar8bits'`): 1-bit storage with an 8-bit query for better precision.

## Multi-vector / ColPali

Qdrant natively supports the MaxSim multi-vector representation [[ColPali]] needs. Store pooled *and* full multivectors as named vectors, then prefetch on pooled and exact-rerank on full — server-side in one Query API call. Their measured result: mean pooling preserved **NDCG@20 = 0.952, Recall@20 = 0.917** with **13× faster** retrieval. Max pooling underperformed.

Note HNSW build cost grows quadratically with vectors per page, so pooling is not optional at scale.

## Also from Qdrant

**miniCOIL** — their own learned-sparse model and current recommendation for new sparse projects. See [[Learned Sparse Retrieval]].

## Related

[[OpenSearch]] · [[Neo4j]] · [[ColPali]] · [[Embedding Quantization and MRL]] · [[Hybrid Retrieval and RRF]]
