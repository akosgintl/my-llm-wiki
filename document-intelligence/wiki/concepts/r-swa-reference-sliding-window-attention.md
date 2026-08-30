---
aliases: ["R-SWA Reference Sliding Window Attention"]
tags: [architecture, attention, long-context, ocr]
sources: [ocr-arxiv-github-technical-review.md, Deep Technical Review - Open-Source OCR - Visual Document Understanding Models - Missed Details and Pipeline Implications.md, Open-Source OCR per Visual Document Understanding Models for Self-Hosted AKS Pipelines - A Comparative Investigation (August 2026).md]
created: 2026-08-27
updated: 2026-08-27
---

# R-SWA Reference Sliding Window Attention

The attention mechanism behind [[Baidu Unlimited-OCR]] (arXiv 2606.23050), and the reason it can parse dozens of pages in one request.

## The mechanism

Every decoder attention layer is replaced. Each generated token attends to:

1. **all reference tokens** — visual tokens + prompt, length **m**, fixed at inference start, held **statically outside the state transitions**; and
2. only the preceding **n = 128** output tokens.

The KV cache is implemented as a **queue of capacity m + n**: each new token evicts the oldest *output* token's KV. Cache ceiling is `C(T) = L·m + min(n, T) ≤ L·m + n` — **constant**, versus O(T) for vanilla attention.

## Why not vanilla SWA or linear attention

In those, **visual tokens get pulled into recursive state updates and the image features progressively blur as pages advance.** Excluding reference tokens from the sliding state is the entire trick — and it is also why the fine-tune recipe freezes the vision encoder.

## What it buys

- **40+ pages in one call** at edit distance <0.11.
- ~35% faster than [[DeepSeek-OCR]] at 6,000 generated tokens (TPS 7,847 vs 5,823), with the gap widening as output grows.
- Distinct-20 = 96.08% / Distinct-35 = 96.90% — the paper's own anti-loop metric. Bounded output attention makes long-horizon looping structurally harder. See [[Repetition Loops in VLM OCR]].
- Flat memory and latency regardless of output length, which is what makes it attractive on MIG-partitioned H100s and 24 GB cards.

## What it does not buy

`max_length = 32768` still bounds **prompt + reference + output together**, so the practical ceiling is ~30–40 text pages and fewer for dense tables. The naive "pages × 800 tokens" heuristic must subtract vision and prompt tokens. Per-request `window_size` must match the serving engine's processor args (128 for gundam, 1024 for multi-page) — the eviction-queue design is why that parameter matters at all.

Serving the long-context path optimally needs **FlashAttention-3, i.e. Hopper-only**. See [[SGLang]].

## Where it is heading

The paper's roadmap states a 128K-context training run and a **"prefill pool"** that fetches document chunks on demand — a human page-flipping analogue — and positions R-SWA as general-purpose for ASR and translation. Practical consequence: **do not over-engineer chunking now**; expect a longer-context v2.

## Related

[[Baidu Unlimited-OCR]] · [[Optical Compression]] · [[Document Fan-Out and Fan-In]] · [[MIG and GPU Sharing]]
