---
aliases: ["DeepSeek-OCR"]
tags: [ocr, vlm, document-parsing, model, licensing]
sources: [Open-Source OCR per Visual Document Understanding Models for Self-Hosted AKS Pipelines - A Comparative Investigation (August 2026).md, ocr-serving-recipes-runpod-v2.md, ocr-api-call-strategy.md, ocr-vdu-complete-study.md, ocr-arxiv-github-technical-review.md, Deep Technical Review - Open-Source OCR - Visual Document Understanding Models - Missed Details and Pipeline Implications.md, Local PDF-Markdown and Document Parsing - State of the Art (August 2026).md]
created: 2026-08-27
updated: 2026-08-27
---

# DeepSeek-OCR

[[DeepSeek AI]]'s optical-compression document model. v1 released Oct 2025 (arXiv 2510.18234), **DeepSeek-OCR 2** on 2026-01-27 (arXiv 2601.20552).

## Architecture

- **DeepEncoder**: SAM-ViT-base (patch-16 window attention) + CLIP-large global, ~380M params, 16× convolutional token compression, feeding a **DeepSeek-3B-MoE** decoder (6 routed + 2 shared experts, ~570M active).
- **OCR-2 / DeepEncoder V2**: a Qwen2-0.5B-based "Visual Causal Flow" module that reorders visual tokens into a learned human reading order before the decoder. Local views share 144 query embeddings; total reordered tokens = k×144+256, range [256, 1120], k crops 0–6. The attention mask is bidirectional (vision, ViT-like) concatenated with causal-triangular (flow tokens); only the causal flow tokens reach the LLM. Decoder unchanged.

## Resolution modes (serving-critical)

| Mode | Size | Vision tokens |
|---|---|---|
| Tiny | 512² | 64 |
| Small | 640² | 100 |
| Base | 1024² | 256 |
| Large | 1280² | 400 |
| Gundam | n×640² tiles + 1×1024² global | n×100 + 256 |
| Gundam-Master | 1024² local + 1280² global | — |

Base/Large pad to aspect ratio, so valid tokens are fewer than allocated. Your raster target must match the chosen mode's native size exactly — see [[Rasterization at Model-Native Resolution]].

## Optical compression

Verbatim from the paper: ~97% decoding precision at <10× text-token-to-vision-token compression, ~90% at 10–12×, ~60% at 20× (Fox benchmark). Reading-order edit distance improved 0.085→0.057 in OCR-2. Throughput ~200,000 pages/day on a single A100-40G. See [[Optical Compression]].

## Prompt grammar (newline-sensitive)

```
<image>\n<|grounding|>Convert the document to markdown.
<image>\nFree OCR.
<image>\nParse the figure.
<image>\nLocate <|ref|>xxx<|/ref|> in the image.
```

The Ollama card warns the model is input-sensitive: **a missing punctuation mark or newline can corrupt output**. Pin prompt strings byte-exactly in the gateway and in CI. Training data includes charts, chemical structures (it can emit SMILES) and geometry.

## Failure modes

- **[[Repetition Loops in VLM OCR]] are the dominant production failure.** Jim Clifford (University of Saskatchewan) reported a persistent **9.2% catastrophic failure cohort** on 600 British Library historical newspaper clippings (GitHub Issue #151). DeepSeek's own OCR-2 paper measures 6.25%→4.17% (user-log images) and 3.69%→2.88% (PDF production). The n-gram logits processor is **not optional**.
- **[[Linguistic Crutch and Faithfulness]]**: the critique paper "Visual Merit or Linguistic Crutch?" (arXiv 2601.03714) shows accuracy plummets from ~90% to ~20% under sentence- and word-level semantic corruption, and that *fewer visual tokens correlate with more reliance on language priors and more hallucination*. Traditional pipeline OCR was significantly more robust across 13 compared baselines.
- Near-bottom on independent multi-script evaluation — v1 scores below Tesseract v5 on socOCRbench. See [[Multi-Script OCR Benchmarks]].

**Read the evidence per version, not per lineage.** OCR-2 supersedes v1 on every published number — olmOCR-Bench 76.3 vs ~75.4–75.7, OmniDocBench overall 73.01, reading-order edit distance 0.085→0.057, repetition 6.25→4.17% — **and is still near the bottom on multi-script** (socOCRbench ~0.176 against [[dots.ocr]] ~0.478; 40+ points below the GlotOCR leader). The 9.2% catastrophic cohort was measured on **v1**; do not carry that number forward to v2, and do not carry v2's improvements backward as if they closed the multi-script gap. This split is why the lineage occupies **two rows** on the [[Hungarian Model Decision Matrix]]: v2 is the fine-tune base to train, v1 is only the historical baseline.

## Licensing wrinkle

v1 code MIT, OCR-2 repo Apache-2.0 — but the toolchain imports **AGPL-3.0 PyMuPDF**. GitHub Issue #223 (opened 2025-11-05) argues the project "becomes a derivative work and would have to be licensed under the same terms." A genuine commercial blocker: replicate rasterization with [[pypdfium2]] rather than vendoring their scripts. See [[Permissive Licensing Constraints]].

## Fine-tuning

The best-supported fine-tune target in the field via [[Unsloth]] — but the **stock checkpoint does not run or train on current transformers**; use the modified `unsloth/DeepSeek-OCR[-2]` upload (Stranger Vision patches). Their Persian demo: 200K samples, 88.26% absolute CER improvement (149.07%→60.81%), most of the gain arriving after 60 steps at batch 8. See [[LoRA Fine-Tuning for OCR]].

## Serving

[[vLLM]] with `--logits-processors vllm.model_executor.models.deepseek_ocr:NGramPerReqLogitsProcessor`, `--no-enable-prefix-caching`, `--mm-processor-cache-gb 0`; per request `vllm_xargs {"ngram_size": 30, "window_size": 90}`. Also Ollama, llama.cpp (GGUF down to ~4 GB Q4), and a Rust multi-backend port (deepseek-ocr.rs).

## Related

[[Baidu Unlimited-OCR]] · [[Optical Compression]] · [[Unsloth]] · [[Confidence Engineering]] · [[vLLM]] · [[Hungarian Model Decision Matrix]] · [[Multi-Script OCR Benchmarks]]
