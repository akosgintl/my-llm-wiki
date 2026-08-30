---
aliases: ["Tiered Page Routing"]
tags: [architecture, pipeline, cost, hungarian, document-parsing]
sources: [tiered-pdf-pipeline-architecture.md, Local PDF-Markdown and Document Parsing - State of the Art (August 2026).md, ocr-vdu-complete-study.md]
created: 2026-08-27
updated: 2026-08-27
---

# Tiered Page Routing

Classify each page and route it to the cheapest tier that can handle it. This is what [[Marker and Chandra]] (device-dependent modes) and [[MinerU]] (pipeline/VLM/hybrid backends) now do internally — and it gives the best cost/quality trade-off available.

## The three tiers

| Tier | Handles | Tool | Cost |
|---|---|---|---|
| **1 — fast text** | clean digital text | [[liteparse]] (Apache-2.0) | CPU, ~ms/page |
| **2 — layout pipeline** | tables, multi-column | [[Docling]] + TableFormer + [[Tesseract]] `hun` | CPU or small GPU |
| **3 — full-page VLM** | scanned, garbled, math | [[PaddleOCR-VL]] via [[vLLM]] | GPU, batched |

All three normalize to **one schema** and then pass through validation:

```python
@dataclass
class Block:
    text: str
    bbox: tuple[float, float, float, float]
    type: BlockType          # text | table | formula | figure | ...
    confidence: float
    source_tier: int         # 1 | 2 | 3
    page_index: int
```

Carry `source_tier` from day one — it is what makes the escalation matrix measurable.

## The router (per page, <1 ms on CPU)

Five signals, **fail 2 of 5 → Tier 3**:

1. text token density
2. bitmap coverage ≥0.95 → scanned
3. embedded fonts? `GlyphlessFont` indicates prior OCR
4. CID-escape ratio >0.5 → garbled
5. **`hu_HU` dictionary ratio + mojibake / Latin-2 check**

Otherwise a [[PP-DocLayout-V3]] pass decides: tables or formulas → Tier 2, else Tier 1.

## The escalation edge

A page that **fails validation is re-routed to the next tier up**, so a bad Tier-1 parse costs one retry rather than a corrupted document. Validation: hunspell `hu_HU` token ratio, diacritic check, table mismatch.

**Primary tuning signal: the escalation matrix (tier 1 → 3 rate).** Escalation above ~30% means the router thresholds are miscalibrated or the corpus is scan-heavier than assumed.

## Hungarian specifics that change the design

1. **Tier 3 model choice is forced.** [[olmOCR]] is English-focused, so it is out. PaddleOCR-VL's 109 languages include Hungarian; [[dots.ocr]] covers it too. Benchmark both on *your* scans — published multilingual scores rarely break out Hungarian.
2. **The double acute is the canary.** Build an eval set dense in ő/ű and make double-acute accuracy an explicit **per-tier** metric. Tesseract `hun` handles it well on clean scans; VLM output on degraded scans is where drift appears.
3. **Validation must be Hungarian-aware.** An English wordlist would flag every correct Hungarian page as garbage and escalate the whole corpus to the GPU.
4. **Legacy encodings route to OCR.** Set `lang=hun` explicitly, and add a Latin-2 / CP-1250 mojibake pattern — older Hungarian PDFs carry broken `õ`-for-`ő` text layers that should skip Tier 1 entirely.

## Licensing exclusions

Permissive-only (MIT / Apache-2.0 / BSD) removes: [[pymupdf4llm]] (AGPL) → [[pypdfium2]] + liteparse · Marker (GPL + OpenRAIL-M) → Docling + PaddleOCR-VL · Surya weights and DocLayout-YOLO (OpenRAIL-M / AGPL) → PP-DocLayout · olmOCR (Apache but English) → PaddleOCR-VL. [[MinerU]] is permissive-ish with attribution conditions — optional, add only if its hybrid mode wins on your corpus. See [[Permissive Licensing Constraints]].

## Rollout

| Stage | Scope |
|---|---|
| 1 | Router + Tier 1: pypdfium2 signals, the 2-of-5 voting rule, liteparse, DoclingDocument normalization with `source_tier` |
| 2 | Tier 2 + validation: PP-DocLayout signal, Docling+TableFormer+Tesseract `hun`, hu_HU garble checks, escalation logging |
| 3 | Tier 3: PaddleOCR-VL on vLLM, async batched page images, temperature-ramp retries, document-level max error rate (~0.004) |
| 4 | Cross-page table/paragraph merge, page-image hash cache, per-tier cost and escalation-rate metrics |

Every slot is swappable behind one `Parser` protocol, and Tier 3 hides behind an OpenAI-compatible endpoint — so changing the VLM is a config change. See [[Pipeline as Platform, Model as Config]].

See [[Cost per Page Model]] for what the escalation rate is worth in dollars — moving it from 100% to 20% saves 5×, more than any model swap.

## Related

[[liteparse]] · [[Docling]] · [[PaddleOCR-VL]] · [[Hungarian OCR and the Double Acute Test]] · [[Born-Digital Self-Labeling]] · [[Permissive Licensing Constraints]]
