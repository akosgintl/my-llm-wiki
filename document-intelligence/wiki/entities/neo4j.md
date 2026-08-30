---
aliases: ["Neo4j"]
tags: [rag, graph, retrieval, infrastructure]
sources: [SOTA Agentic RAG Reference Architecture - A System-Design Session Brief.md, The Everything Else Map of SOTA RAG Optimization (2024–2026) - A Decision-Ready Survey for a Greenfield Agentic RAG System.md, Deep-Dive - 10 RAG Optimization Techniques for a Self-Hosted Agentic Stack.md]
created: 2026-08-27
updated: 2026-08-27
---

# Neo4j

The graph store in the RAG architecture — used as a **lightweight document/lexical graph plus an optional entity graph**, deliberately not as a heavyweight ontology. See [[GraphRAG and Document Graphs]].

## Schema

```cypher
CREATE CONSTRAINT doc_id   IF NOT EXISTS FOR (d:Document) REQUIRE d.id IS UNIQUE;
CREATE CONSTRAINT sec_id   IF NOT EXISTS FOR (s:Section)  REQUIRE s.id IS UNIQUE;
CREATE CONSTRAINT chunk_id IF NOT EXISTS FOR (c:Chunk)    REQUIRE c.id IS UNIQUE;
CREATE FULLTEXT INDEX chunkText IF NOT EXISTS FOR (c:Chunk) ON EACH [c.text];
```

```
(:Document {id, title, source_uri, lang, created_at})
  -[:HAS_SECTION]->(:Section {id, title, order})
  -[:HAS_CHUNK]->(:Chunk {id, text, embedding, token_count, order, content_hash})

(:Chunk)-[:NEXT]->(:Chunk)                   // sequential order for context expansion
(:Chunk)-[:LINKS_TO]->(:Document|:Section)    // cross-references — the reference paper's key edge

// entity extension
(:Chunk)-[:MENTIONS]->(:Entity {id, name, type, aliases})
(:Entity)-[:RELATED_TO {type, evidence_chunk_id}]->(:Entity)
(:Entity)-[:IN_COMMUNITY]->(:Community {id, summary, level})
```

`content_hash` drives incremental re-indexing. [[BGE-M3]] and multilingual-e5-large both produce 1024-dim vectors.

## Vectors: keep them in Qdrant

Neo4j's vector index is an add-on to a graph engine; benchmarks show dedicated stores deliver **~10× lower vector latency**. The proven pattern (Lettria's GraphRAG at 100M+ embeddings, sub-200 ms, reported +20–25% accuracy) keeps **[[Qdrant]] for vectors + Neo4j for relationships, joined by shared IDs**, with the Neo4j commit as the consistency gate. Use the built-in vector index only for small corpora or to avoid a second system in an MVP.

## Expose curated Cypher, not text2cypher

Give the agent **parameterized Cypher tools** — `get_neighbors(chunk_id)`, `expand_section(section_id)`, `entities_between(a, b)` — rather than letting it generate raw Cypher. See [[Text2Cypher]] for why.

## First-party tooling

`neo4j-graphrag-python` (Apache-2.0, long-term support) with `SimpleKGPipeline` (parse → chunk → embed → schema-guided entity/relation extraction → entity resolution → write), plus the LLM Knowledge Graph Builder app. Natively supports Vector, Hybrid, GraphRAG and Text2Cypher retrievers, and ships a **native Qdrant retriever**. Entity resolution via `SpaCySemanticMatchResolver` and `FuzzyMatchResolver` (RapidFuzz).

## License caution

Neo4j Community is **GPLv3**; Enterprise features are commercial. See [[Permissive Licensing Constraints]].

## Also lands here

A [[RAPTOR Hierarchical Summarization]] tree maps naturally onto Neo4j nodes (`:Chunk`/`:Summary` with `PARENT_OF` edges), which unifies auto-merging and RAPTOR retrieval into graph traversals and lets you localize tree rebuilds to affected subgraphs. Graphiti (Apache-2.0) can sit on the same Neo4j for temporal agent memory.

## Related

[[Qdrant]] · [[GraphRAG and Document Graphs]] · [[Text2Cypher]] · [[RAPTOR Hierarchical Summarization]]
