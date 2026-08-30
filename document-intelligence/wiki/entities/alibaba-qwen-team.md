---
aliases: ["Alibaba Qwen Team"]
tags: [organization]
sources: [ocr-language-models-deployment-critical-pass-2.md, ocr-vdu-complete-study.md, ocr-arxiv-github-technical-review.md, Deep-Dive - 10 RAG Optimization Techniques for a Self-Hosted Agentic Stack.md, The Everything Else Map of SOTA RAG Optimization (2024–2026) - A Decision-Ready Survey for a Greenfield Agentic RAG System.md]
created: 2026-08-27
updated: 2026-08-27
---

# Alibaba Qwen Team

The organization whose output touches the most slots in this architecture, on both the OCR and the RAG side:

| Slot | Model |
|---|---|
| Recognition (behind the [[glmocr SDK]]) | [[Qwen Model Family]] — **Qwen3.5-9B** default, 4B as the throughput arm |
| Dense embeddings | [[Qwen3-Embedding]] (0.6B/4B/8B, Apache-2.0) |
| Reranking (upgrade path) | Qwen3-Reranker (0.6B/4B/8B, Apache-2.0) |
| Small-model utility work | Qwen 3 4B for conversational rewrite, Self-Query extraction, decomposition |
| Foundations of other models | Qwen2.5-1.5B under [[dots.ocr]], Qwen2-0.5B under [[MinerU]] and DeepEncoder V2, Qwen2.5-VL-7B under [[olmOCR]] |
| Vision generation over page images | Qwen2.5-VL on [[vLLM]] for [[ColPali]] retrieval results |

Apache-2.0 across the line, which is why the permissive-only constraint keeps landing on them.

## The version treadmill

They ship a generation roughly every two months (Qwen3.5 → 3.6 → 3.7 → 3.8 within 2026), and most of those releases are **irrelevant** to document work — 3.6 and beyond are coding-focused, drop the small sizes, and carry no OCR evidence.

The cure is architectural rather than editorial: because the recognition model is a config value, any new checkpoint is **one `SERVED_NAME` change plus one golden-set run away from a verdict**. Adoption is earned on the [[Golden Set and Eval Harness]], never on the release blog. See [[Pipeline as Platform, Model as Config]].

## Related

[[Qwen Model Family]] · [[Qwen3-Embedding]] · [[Pipeline as Platform, Model as Config]]
