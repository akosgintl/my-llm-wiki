---
aliases: ["Indirect Prompt Injection"]
tags: [rag, security, reliability, compliance]
sources: [The Everything Else Map of SOTA RAG Optimization (2024–2026) - A Decision-Ready Survey for a Greenfield Agentic RAG System.md, Deep-Dive - 10 RAG Optimization Techniques for a Self-Hosted Agentic Stack.md]
created: 2026-08-27
updated: 2026-08-27
---

# Indirect Prompt Injection

**OWASP LLM01:2025 — the #1 ranked LLM risk.** RAG and fine-tuning do not eliminate it. For any corpus that ingests untrusted content, indirect injection via poisoned retrieved documents is not a hypothetical.

## The threat model, quantified

**AgentPoison** (Chen et al., NeurIPS 2024, arXiv 2407.12784) backdoors an agent's memory or RAG store via optimized triggers:

> *"AgentPoison achieves an average attack success rate of ≥ 80% with minimal impact on benign performance (≤ 1%) with a poison rate < 0.1%"*

— and **a single poisoning instance with a single-token trigger reaches ≥60% ASR, transferable across model families**.

**PoisonedRAG** (arXiv 2402.07867, USENIX Security 2025) injects a few malicious texts to force an attacker-chosen answer. Indirect injection can also exfiltrate data or trigger tools.

Benchmarks for evaluating this: **InjecAgent** and **AgentDojo** (NeurIPS 2024).

## Layered defenses

**1. Architectural isolation — the most durable, because it is model-independent.** CaMeL (Google DeepMind, arXiv 2503.18813) extracts control and data flow from the *trusted* query via a custom Python interpreter, so **untrusted retrieved data can never alter program flow**; capability-based policies prevent exfiltration. Reported: solves 67% (v1) / 77% (v2) of AgentDojo tasks *with provable security*, versus 84% undefended and no guarantees. Dual-LLM patterns are a lighter cousin.

**2. Spotlighting / delimiting** (Microsoft, arXiv 2403.14720) — transform untrusted input (delimiting, datamarking, encoding) to give the model a continuous provenance signal separating data from instructions. This is the same reason XML-style context delimiters are preferred — see [[Context Pruning and Lost-in-the-Middle]].

**3. Instruction hierarchy** (arXiv 2404.13208) — train or prompt the model to prioritize privileged system/developer instructions over retrieved-content instructions.

**4. Hybrid search + reranking raise the bar** — an attacker must beat both the lexical and the semantic stage. **You already have this** if you run [[Hybrid Retrieval and RRF]] and [[Two-Stage Retrieve-Then-Rerank]]; it is defense you get for free.

**5. Ingestion-time sanitization** — strip or neutralize instruction-like content, HTML, hidden text and zero-width characters at index time. CleanBase-style malicious-document detection; CommandSans (arXiv 2510.08829) for surgical sanitization.

**6. Detection classifiers** — ProtectAI `deberta-v3-base-prompt-injection-v2` (verify Apache-2.0 on the card) as a [[LangGraph]] guard node on both the query and the retrieved content. **Meta PromptGuard 2 is Llama-licensed, not permissive** — see [[Permissive Licensing Constraints]].

**7. Canary tokens** to detect exfiltration, plus output filtering.

## The v1 shortlist

Start with **ingestion sanitization + spotlighting** (cheap, high value), add the permissive DeBERTa guard node, and **scope LangGraph tool capabilities so retrieved content cannot parameterize dangerous tools**. This is also why [[Text2Cypher]] is discouraged in favor of curated Cypher tools.

## Adjacent: ACL-aware retrieval

Filter documents by access-control metadata **at retrieval time** — per user, tenant or namespace — before they reach the LLM. **Never let a semantic cache or prefix cache cross tenant boundaries** ([[vLLM]] supports per-request cache salting for isolation). PII detection and redaction at ingestion.

## Honest limit

**No single defense is complete.** Adaptive attackers bypass fine-tuning-based defenses (arXiv 2507.07417) and even layered prompting under domain camouflage (arXiv 2606.18530). CaMeL degrades utility. Treat defense-in-depth as risk *reduction*, not elimination.

## Related

[[Hallucination Detection in RAG]] · [[Text2Cypher]] · [[LangGraph]] · [[Permissive Licensing Constraints]]
