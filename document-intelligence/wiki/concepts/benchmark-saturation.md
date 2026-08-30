---
aliases: ["Benchmark Saturation"]
tags: [benchmarks, evaluation, methodology]
sources: [Open-Source OCR per Visual Document Understanding Models for Self-Hosted AKS Pipelines - A Comparative Investigation (August 2026).md, ocr-vdu-complete-study.md, Local PDF-Markdown and Document Parsing - State of the Art (August 2026).md, Deep Technical Review - Open-Source OCR - Visual Document Understanding Models - Missed Details and Pipeline Implications.md]
created: 2026-08-27
updated: 2026-08-27
---

# Benchmark Saturation

The dominant document-parsing benchmarks are **saturated, vendor-reported, and biased toward clean Latin/CJK print**. Reading them as a ranking produces wrong decisions.

## The three failures

**1. Compression.** Top open models on [[OmniDocBench]] v1.6 cluster within ~2 points at 94–96. LlamaIndex has publicly called it saturated. A 1–2 point delta is noise.

**2. Self-reporting.** Vendors typically self-evaluate while pulling competitor numbers from the benchmark repo. Both [[OpenDataLab]] (MinerU + OmniDocBench) and [[Allen Institute for AI]] (olmOCR + olmOCR-Bench) publish a leading model *and* the benchmark it tops. Some aggregators also inflate scores using quantized variants.

**3. Metric artifacts.** OmniDocBench's edit distance is formatting-sensitive, which is part of why a structurally sound framework like [[Docling]] scores badly (0.589 EN / 0.909 ZH). olmOCR-Bench's header/footer category can reward *omitting* content.

## The inversion, in numbers

The same models, ranked by vendor benchmark versus by independent evaluation:

| Model | OmniDocBench v1.6 | socOCRbench | [[OCR Arena]] ELO |
|---|---|---|---|
| [[PaddleOCR-VL]]-1.6 | **96.33** (1st) | ~0.394 (2nd) | — |
| [[MinerU]]2.5-Pro | 95.69 (2nd) | ~0.165 (near-last) | — |
| [[GLM-OCR]] | ~95.2 (3rd) | ~0.368 (3rd) | 1347 (**#24**) |
| [[dots.ocr]] | ~88.4 (low) | **~0.478 (1st)** | 1442 (**#12**) |
| [[DeepSeek-OCR]] v1 | 87.01 | ~0.086 (**below Tesseract**) | historically last |

GLM-OCR tops OmniDocBench and sits at #24 on blind human preference, losing to dots.ocr 88.9% of the time. On GlotOCR it and DeepSeek-OCR-2 sit **over 40 points below** Gemini 3.1 Flash-Lite.

## The contested-evaluation problem is not OCR-specific

The same pattern appears in RAG. A 2025 study found reported GraphRAG gains partly stem from **LLM-judge position, length and verbosity biases** that shrink or vanish when corrected. Independent work (GroUSE, arXiv 2409.06595) shows LLM-judge faithfulness scorers have blind spots. And "Scaling RAG with RAG Fusion" found that recall gains from multi-query fusion were "largely neutralized after re-ranking and truncation" — a technique that benchmarks well in isolation and disappears in a full pipeline.

## What to do instead

1. **Build a [[Golden Set and Eval Harness]] on your own corpus.** It is the only leaderboard that counts.
2. **Read the independent evaluations to narrow the field**, not to rank it — see [[Multi-Script OCR Benchmarks]].
3. **Never trust a leaderboard delta under ~2 points.** Use confidence intervals at realistic sample sizes; many reported gains vanish under proper CIs.
4. **Re-validate quarterly.** New SOTA claims land roughly monthly and any "leader" claim has a short shelf life. Architect for swappable backends — see [[Pipeline as Platform, Model as Config]].

## Related

[[OmniDocBench]] · [[olmOCR-Bench]] · [[OCR Arena]] · [[Multi-Script OCR Benchmarks]] · [[Golden Set and Eval Harness]] · [[RAG Evaluation]]
