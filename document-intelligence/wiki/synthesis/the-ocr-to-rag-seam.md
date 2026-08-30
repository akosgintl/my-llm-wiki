---
aliases: ["The OCR-to-RAG Seam"]
tags: [synthesis, architecture, rag, ocr, chunking, retrieval, decision]
sources: [tiered-pdf-pipeline-architecture.md, SOTA Agentic RAG Reference Architecture - A System-Design Session Brief.md, The Everything Else Map of SOTA RAG Optimization (2024–2026) - A Decision-Ready Survey for a Greenfield Agentic RAG System.md, Deep-Dive - 10 RAG Optimization Techniques for a Self-Hosted Agentic Stack.md, ocr-vdu-complete-study.md]
created: 2026-08-27
updated: 2026-08-27
---

# The OCR-to-RAG Seam

The corpus is two halves — eleven documents designing a PDF→markdown pipeline, then a hard pivot to retrieval — and **the join between them is never specified anywhere**. The RAG brief reserves a slot for "the OCR pipeline"; the OCR documents end at a markdown file. This page is that slot, filled in.

The seam matters more than either side's internals, because **the two halves were designed by different documents and their contracts do not automatically line up.**

## What the OCR side actually emits

[[Tiered Page Routing]] normalises all three tiers to one `Block` schema. Four of its fields are the ones that must survive into retrieval:

| Field | Why RAG needs it |
|---|---|
| `source_tier` (1/2/3) | **the single most valuable retrieval-time signal you already have and would otherwise throw away** |
| `confidence` | gates whether a chunk may be cited without a human in the loop |
| `bbox` + `page_index` | the only way to render a citation back onto the original page |
| `type` (text/table/formula/figure) | tables must not be chunked like prose |

**The default failure is that all four are dropped at the markdown boundary.** Markdown is the canonical *content* format; it is not a transport for provenance. Carry these as chunk metadata alongside the text, not inside it.

**A second failure became possible on 2026-08-27.** `bbox` and `page_index` are only free if the recognition model emits them. [[Qwen Model Family]] Qwen3-VL documents QwenVL-HTML with `data-bbox` attributes; **the Qwen3.5, 3.6 and 3.8 repos document no such output**, and Qwen3.5 is where the usable model sizes are. If the model in the recognition slot cannot emit coordinates, they have to come from the layout stage ([[PP-DocLayout-V3]]) instead — which is an argument for keeping the layout detector even where an end-to-end model would otherwise replace it. Verify before designing around it; it is listed in [[Open Inputs and Corpus Profile]].

## The four decisions the seam forces

### 1. Chunk on structure, not on the markdown string

[[Chunking Strategies]] says chunking influences retrieval quality as much as the embedding model, and parent-child is the default. The OCR side already gives you the structure for free — layout blocks and reading order from [[PP-DocLayout-V3]]. **Chunk from the block tree, not by re-parsing the emitted markdown.** Re-parsing markdown to recover structure the layout stage already computed is the most common and most avoidable loss at this seam.

Practical rule: **block = child, section = parent.** A table is one atomic child regardless of length.

### 2. `source_tier` becomes a retrieval filter and a rerank feature

A Tier-1 chunk (clean born-digital text) and a Tier-3 chunk (VLM output on a bad scan) have **materially different reliability**, and nothing downstream can tell them apart once they are both strings in a vector store. Carry the tier and use it:

- as a **metadata filter** for high-stakes queries — restrict to Tier 1–2 when an exact figure is being asked for;
- as a **rerank tiebreaker** in [[Two-Stage Retrieve-Then-Rerank]] when scores are close;
- as the **grader input** in [[Corrective RAG]] — a low-confidence Tier-3 chunk should trip the corrective branch sooner than a Tier-1 chunk with the same relevance score.

This costs nothing at ingest and cannot be reconstructed later.

### 3. [[Contextual Retrieval]] is where the two halves actually pay off together

The highest-ROI ingestion technique on the RAG side needs **document context** to prepend to each chunk. The OCR side is the only thing that has it — section headings, document type, page position — and it has it in structured form before markdown flattens it. Running Contextual Retrieval *after* discarding the block tree means re-deriving with an LLM what the layout model already knew.

### 4. Decide the [[ColPali]] question at the seam, not later

[[The OCR-to-Text Boundary Limit]] is a measured cliff: **CharXiv 0.0 for two-stage OCR + text LLM against 81.8–94.0 end-to-end.** Charts, checkboxes, stamps and spatial relationships do not survive the seam at all — no chunking strategy recovers them.

The pipeline already rasterises every page at model-native resolution ([[Rasterization at Model-Native Resolution]]), so ColPali's expensive prerequisite is **already paid**. The decision threshold is concrete: *if OCR output already answers >~95% of table/chart questions correctly, defer ColPali.* That is a golden-set measurement, and it is the same golden set as [[Golden Set and Eval Harness]] — one artifact, both halves.

**Storage caveat if you do adopt it:** ~1,030 vectors per page, so token pooling plus two-stage rerank is mandatory, not optional.

## The contract, stated once

```python
@dataclass
class Chunk:
    text: str                 # markdown, canonical content
    parent_id: str | None     # section-level parent for parent-child retrieval
    doc_id: str
    page_index: int
    bbox: tuple[float, float, float, float]
    block_type: BlockType     # text | table | formula | figure
    source_tier: int          # 1 | 2 | 3   <- from the OCR router
    ocr_confidence: float     # <- from the OCR validator
    context_prefix: str       # <- Contextual Retrieval, generated from the block tree
```

Everything above the line is content; everything below it is provenance. **Both go into the vector store; only `text` and `context_prefix` go into the embedding.**

## Why this page exists

Neither half of the corpus is wrong — but the OCR documents optimise for *fidelity to the page* and the RAG documents optimise for *retrievability of the chunk*, and those are different objectives that meet exactly here. [[Pipeline as Platform, Model as Config]] applies to the seam too: **the contract above is the fixed point**, and both the OCR model and the embedding model are config values behind it.

## Related

[[Tiered Page Routing]] · [[The OCR-to-Text Boundary Limit]] · [[Chunking Strategies]] · [[Contextual Retrieval]] · [[ColPali]] · [[Two-Stage Retrieve-Then-Rerank]] · [[Corrective RAG]] · [[Confidence Engineering]] · [[Golden Set and Eval Harness]]
