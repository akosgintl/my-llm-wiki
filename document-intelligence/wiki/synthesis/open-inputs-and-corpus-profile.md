---
aliases: ["Open Inputs and Corpus Profile"]
tags: [synthesis, methodology, decision, open-questions]
sources: [ocr-vdu-complete-study.md, tiered-pdf-pipeline-architecture.md, Deep Technical Review - Open-Source OCR - Visual Document Understanding Models - Missed Details and Pipeline Implications.md, ocr-arxiv-github-technical-review.md, SOTA Agentic RAG Reference Architecture - A System-Design Session Brief.md, The Everything Else Map of SOTA RAG Optimization (2024–2026) - A Decision-Ready Survey for a Greenfield Agentic RAG System.md]
created: 2026-08-27
updated: 2026-08-27
---

# Open Inputs and Corpus Profile

Everything still blocking a decision, in one place, sorted by **who can answer it**. The wiki holds 116 pages of research and it is not the research that is missing — **it is five facts about your own corpus that only you have**, plus a short tail of external checks.

The distinction matters because the two kinds fail differently: an external gap is a search away, while a corpus gap **cannot be closed by more reading, no matter how much you do.**

## Blocking — only you can answer these

Every one of these is named as "still open" by the sources, and each independently gates a decision that is otherwise fully researched.

| Input | What it decides | Blocked without it |
|---|---|---|
| **Volume** — pages/month, and peak vs average | GPU count, reserved vs spot, self-host vs API crossover | [[Cost per Page Model]] is a rate with no quantity |
| **Born-digital vs scanned split** | the escalation rate, which is the **dominant cost lever** (5× swing) | [[Tiered Page Routing]] cannot be sized |
| **Page-length distribution** | request shape — per-page fan-out vs whole-document | [[Document Fan-Out and Fan-In]]; the [[Baidu Unlimited-OCR]] long-context case |
| **Latency class** — interactive, batch, or overnight | continuous-batching concurrency, MIG slicing, whether Tier 3 may block | [[vLLM Continuous Batching and Concurrency Sizing]], [[MIG and GPU Sharing]] |
| **Data residency and sovereignty requirement** | whether US-incorporated clouds are eligible at all | [[EU Sovereign GPU Hosting]] — CLOUD Act follows incorporation, not geography |

**And the artifact that unblocks the most at once:** 200–500 pages of your own documents, transcribed by hand, with diacritic-specific error rate broken out from overall CER. That is Rule 0 — see [[Golden Set and Eval Harness]]. It converts the entire [[Hungarian Model Decision Matrix]] from a table of priors into a table of results, and it is simultaneously the RAG side's eval set ([[RAG Evaluation]]). **One artifact, both halves of the stack.**

## Externally answerable — the short tail

Checked 2026-08-27; what remains is genuinely small.

| Question | Status | Cost to close |
|---|---|---|
| Does [[Qwen Model Family]] Qwen3.5 still emit QwenVL-HTML with `data-bbox`? | **Open, and newly load-bearing** — documented for Qwen3-VL, **absent from the 3.5/3.6/3.8 repos**. Two of the four fields [[The OCR-to-RAG Seam]] requires depend on it | ~1 hour — prompt 3.5-9B for a layout parse and inspect the output |
| Does the MTP head survive a LoRA merge on [[GLM-OCR]]? | **Still vendor-undocumented**, but narrowed: heads are in-checkpoint, so the realistic failure is a degraded acceptance rate, and gated LoRA is a documented mitigation | ~1 hour — read vLLM's draft acceptance rate from `/metrics` after export |
| Is Hungarian in MORE-149? ([[HunyuanOCR]]'s multilingual claim) | Open | benchmark repo read |

| Is Hungarian among MDPBench's 17 languages? | Open — **new 2026-08-27**. If yes, it is the only real-world multilingual parsing benchmark with direct Hungarian evidence | paper read; see [[Multi-Script OCR Benchmarks]] |
| Is the [[HunyuanOCR]] NOASSERTION licence usable commercially? | Open — **legal, not technical** | counsel review; blocks the most faithful model measured |

**Closed on 2026-08-27** (recorded here so they are not re-opened): Hungarian is **not** among `opensearch-neural-sparse-encoding-multilingual-v1`'s 15 languages ([[Learned Sparse Retrieval]]); Hungarian is **not** declared by [[LightOnOCR]] (11 languages, no CEE language); the [[GLM-OCR]] CogViT connector downsamples **4×** (`spatial_merge_size: 2` over `patch_size: 14`); Qwen3-VL's "32 languages" in the docs is the **>70%-accuracy count**, not the support count — 39 are supported.

**Also closed on 2026-08-27, by reading the primary PDFs rather than searching for them:**

- **[[PaddleOCR-VL]]'s Hungarian claim holds — as membership, not as accuracy.** Re-read from arXiv 2510.14528, Appendix B, Table A1: Hungarian is named in the 47-language Latin group. It is a text table, so this is a hard verification. **What does not exist is any Hungarian-specific accuracy number** — the paper's per-language table scores by *script group* (Latin 0.013) on Baidu's own 107k-sample in-house set, and a Latin aggregate dominated by French and German is precisely where a double-acute failure hides. The roster is also from the **v1 paper**; the 1.6 report never mentions languages, so 1.5/1.6 inherit it rather than reconfirm it. **The residual question is no longer "is Hungarian supported" but "what is the ő/ű error rate" — and only the golden set answers that.**

- **Hungarian is not among Qwen3-VL's 32 above-70% OCR languages.** Figure 2 of arXiv 2511.21631 plots exactly 32 bars, all read at 600 dpi; the 7 supported languages below the threshold are named nowhere in the paper, and the word "Hungarian" does not occur in it. Either branch eliminates Qwen3-VL as a zero-shot Hungarian arm. Full roster in [[Hungarian OCR and the Double Acute Test]].
- **The post-3.5 Qwen generations are real, multimodal, and mostly unusable here — on size, not capability.** Qwen3.5 publishes document benchmarks where 4B matches Qwen3-VL-30B and 9B beats it; but 0.8B–9B open weights exist **only** in 3.5, Qwen3.6 and Qwen3.8 start at 27B dense, and Qwen3.7 never shipped weights. See [[Qwen Model Family]].

## Depth the wiki still owes itself

Not blocking, but thin relative to how load-bearing it is:

- **Eval-harness implementation.** [[Golden Set and Eval Harness]] is Rule 0 and is described as a *discipline*, not as code. The concrete parts — CER/TableTEDS libraries, the diacritic-specific metric, `hu_HU` hunspell wiring, the mojibake regex — are named across four pages and implemented on none. This is the gap most likely to stall execution.
- **Failure semantics under partial failure.** [[Document Fan-Out and Fan-In]] covers the happy path; what a 200-page document does when page 143 times out is under-specified.
- **Reranker and embedding cost.** The RAG half has no equivalent of [[Cost per Page Model]] — [[Two-Stage Retrieve-Then-Rerank]] is justified on quality, never on dollars.

## How to read this page

If an item here is unanswered, **the corresponding decision elsewhere in the wiki is a recommendation, not a conclusion.** That is the honest state of it. The research is done; the measurement is not.

## Related

[[Golden Set and Eval Harness]] · [[Hungarian Model Decision Matrix]] · [[Cost per Page Model]] · [[The OCR-to-RAG Seam]] · [[Tiered Page Routing]] · [[Benchmark Saturation]] · [[Confidence Engineering]]
