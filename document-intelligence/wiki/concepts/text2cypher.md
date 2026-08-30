---
aliases: ["Text2Cypher"]
tags: [rag, graph, security, retrieval]
sources: [SOTA Agentic RAG Reference Architecture - A System-Design Session Brief.md]
created: 2026-08-27
updated: 2026-08-27
---

# Text2Cypher

Letting an LLM generate Cypher against your graph. **"The most flexible, but also the most unreliable" pattern** — and a security surface, not just a reliability one.

## The state of the art

[[Neo4j]] released the **Text2Cypher-2025v1 dataset** (~44k instances) and fine-tuned models (e.g. Gemma-3-27B on HF). Fine-tuned open models get **~40% improvement** on Text2Cypher tests, and one GRPO-refined small model hit **85% execution accuracy on unseen schemas**.

But a 2026 study found the Neo4j Gemma-3-27B model **unsatisfactory on complex schemas** — it *"consistently selected incorrect relationship and node labels."* Accuracy that holds on toy schemas does not survive a real one.

## The recommendation

**Prefer curated parameterized Cypher tools for the core**, as the reference paper does. Expose named, bounded operations:

```
get_neighbors(chunk_id)
expand_section(section_id)
entities_between(a, b)
```

The reference paper's stated reason is explicit: this *"relieve[s] the agent from the task of generating valid data queries – a potential source of failure and security vulnerability."* Both halves matter — a generated query can be wrong, and it can also be an injection vector when the retrieved content that shaped it was untrusted. See [[Indirect Prompt Injection]].

## If you enable it anyway

Layer three defenses:

1. **Schema filtering** — expose only the labels and relationships the agent legitimately needs.
2. **Regex / CyVer validation** before execution.
3. **A ReAct correction loop** so a failed query is repaired rather than returned.

## Related

[[GraphRAG and Document Graphs]] · [[Neo4j]] · [[Indirect Prompt Injection]]
