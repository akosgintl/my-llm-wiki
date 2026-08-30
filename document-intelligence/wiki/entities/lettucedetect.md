---
aliases: ["LettuceDetect"]
tags: [rag, hallucination, groundedness, hungarian, model]
sources: [The Everything Else Map of SOTA RAG Optimization (2024–2026) - A Decision-Ready Survey for a Greenfield Agentic RAG System.md, Deep-Dive - 10 RAG Optimization Techniques for a Self-Hosted Agentic Stack.md]
created: 2026-08-27
updated: 2026-08-27
---

# LettuceDetect

MIT-licensed, ModernBERT-based **token-level groundedness detector** (arXiv 2502.17125), from Ádám Kovács (Budapest University of Technology and Economics) and Gábor Recski (TU Wien), KRLabs.

An unusually good fit for a Hungarian+English corpus — the authorship is Budapest/Vienna and the v2 line explicitly covers Hungarian.

## How it works

Token classification over (context, question, answer) triples, flagging **unsupported answer spans**. Encoder-based, long context up to 8k (trained at 4k), ~30× smaller than the best LLM judges.

## Numbers (RAGTruth)

| System | F1 |
|---|---|
| **lettucedetect-large-v1** | **79.22%** |
| fine-tuned LLAMA-2-13B | 78.7% |
| Luna (previous encoder SOTA) | 65.4% |
| GPT-4 (prompt-based) | 63.4% |

Throughput: **30–60 samples/s on a single GPU**.

## v2 / multilingual

`lettucedect-v2-mmbert-base` (fast encoder) and `lettucedect-v2-qwen-2b` (generative, typed spans) cover code, tool output and prose across the **14-language PsiloQA set including Hungarian**.

## Integration

A post-generation [[LangGraph]] node over (retrieved context, question, draft answer). If unsupported spans exceed a threshold, branch to (a) regenerate constrained to supported content, (b) re-retrieve, or (c) abstain and flag. Because it returns **token spans**, you can surface the exact unsupported text to users or force citations for flagged spans — which pairs directly with chunk-ID citations. See [[Context Pruning and Lost-in-the-Middle]].

```
pip install lettucedetect   # load lettucedect-v2-mmbert-base
```

Set the abstention threshold on a labeled dev set.

## Limits

Trained primarily on the RAGTruth distribution; **an independent 2026 benchmark found lettucedetect-large weak on code-agent hallucinations (0.17 span-F1)** — scope it to prose and document QA. Hungarian performance via PsiloQA should be validated locally.

**Alternatives:** MiniCheck (arXiv 2404.10774; MiniCheck-FT5 770M reaches GPT-4-level at 400× lower cost — English; Bespoke-MiniCheck-7B is stronger but check its custom license); Lynx; Vectara HHEM-2.1 (English/French/German only — no Hungarian). For permissive + Hungarian, LettuceDetect v2 is the best fit.

## Related

[[Hallucination Detection in RAG]] · [[LangGraph]] · [[RAG Evaluation]]
