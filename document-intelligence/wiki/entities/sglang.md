---
aliases: ["SGLang"]
tags: [serving, inference, infrastructure, tool]
sources: [ocr-serving-recipes-runpod-v2.md, ocr-vdu-complete-study.md, ocr-arxiv-github-technical-review.md]
created: 2026-08-27
updated: 2026-08-27
---

# SGLang

The secondary serving engine alongside [[vLLM]]. Relevant to this architecture in two specific places, not as a general alternative.

## Where it matters

**1. [[GLM-OCR]] speculative decoding.** SGLang's `NEXTN` is the equivalent of vLLM's MTP path:

```bash
SGLANG_ENABLE_SPEC_V2=1 sglang serve --model-path zai-org/GLM-OCR \
  --speculative-algorithm NEXTN \
  --speculative-num-steps 3 --speculative-eagle-topk 1 --speculative-num-draft-tokens 4
```

**2. [[Baidu Unlimited-OCR]]'s long-context path — and this is the real reason it appears at all.** The R-SWA constant-KV benefit needs `--attention-backend fa3`, i.e. **FlashAttention-3, which is Hopper-only (H100/H200)**. It also requires `--page-size 1`, `--enable-custom-logit-processor`, `--disable-overlap-schedule`, and **the dev-build SGLang wheel bundled in the repo** — pin it in your image rather than `pip install sglang` fresh. On non-Hopper GPUs, drop the FA3 flag and accept a slower path, or use the vLLM route.

That FA3/Hopper requirement plus the bundled dev-build wheel are genuine AKS deployment friction, and are part of why Unlimited-OCR is quarantined in its own pool.

## Elsewhere

[[PaddleOCR-VL]], [[DeepSeek-OCR]] and [[dots.ocr]] are all servable on SGLang, but vLLM is the better-trodden reference path for each (the official DeepSeek recipe lives at recipes.vllm.ai). Treat SGLang as secondary there and validate output parity before trusting it. [[MinerU]] can drive an SGLang engine internally via `-b vlm-sglang-engine`.

## Related

[[vLLM]] · [[Baidu Unlimited-OCR]] · [[R-SWA Reference Sliding Window Attention]] · [[Multi-Token Prediction and Speculative Decoding]]
