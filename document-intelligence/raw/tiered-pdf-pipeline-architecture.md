# Tiered PDF → Markdown pipeline — reference architecture

Self-hosted, permissive-license-only (MIT / Apache-2.0 / BSD), pluggable, Hungarian-first.

## Architecture

```mermaid
flowchart TD
    A["Ingest & page split<br/><i>pypdfium2 · BSD/Apache</i>"] --> B["Per-page router<br/><i>signals + PP-DocLayout</i>"]

    B -->|"clean digital text"| T1["Tier 1 · fast text<br/><i>liteparse · Apache-2.0</i><br/>CPU, ~ms/page"]
    B -->|"tables / multi-column"| T2["Tier 2 · layout pipeline<br/><i>Docling + TableFormer + Tesseract hun</i><br/>CPU or small GPU"]
    B -->|"scanned / garbled / math"| T3["Tier 3 · full-page VLM<br/><i>PaddleOCR-VL via vLLM</i><br/>GPU, batched"]

    T1 --> N["Normalize<br/><i>DoclingDocument schema</i><br/>block: text, bbox, type,<br/>confidence, source_tier"]
    T2 --> N
    T3 --> N

    N --> V["Validate<br/><i>hunspell hu_HU token ratio ·<br/>diacritic check · table mismatch</i>"]

    V -->|"pass"| M["Cross-page merge<br/><i>table/paragraph continuation</i>"]
    V -.->|"fail → escalate ↻"| B

    M --> O["Markdown / JSON output"]

    style A fill:#F1EFE8,stroke:#5F5E5A,color:#2C2C2A
    style O fill:#F1EFE8,stroke:#5F5E5A,color:#2C2C2A
    style B fill:#EEEDFE,stroke:#534AB7,color:#26215C
    style N fill:#EEEDFE,stroke:#534AB7,color:#26215C
    style V fill:#EEEDFE,stroke:#534AB7,color:#26215C
    style M fill:#EEEDFE,stroke:#534AB7,color:#26215C
    style T1 fill:#E1F5EE,stroke:#0F6E56,color:#04342C
    style T2 fill:#E1F5EE,stroke:#0F6E56,color:#04342C
    style T3 fill:#FAECE7,stroke:#993C1D,color:#4A1B0C
```

The dashed edge is the escalation path: a page that fails validation is re-routed to the next tier up, so a bad tier-1 parse costs one retry, not a corrupted document.

### Router decision signals (per page, <1 ms on CPU)

```mermaid
flowchart LR
    P["Page"] --> S1["Text token density"]
    P --> S2["Bitmap coverage<br/>≥0.95 → scanned"]
    P --> S3["Embedded fonts?<br/>GlyphlessFont = prior OCR"]
    P --> S4["CID-escape ratio<br/>>0.5 → garbled"]
    P --> S5["hu_HU dictionary ratio<br/>+ mojibake / Latin-2 check"]
    S1 & S2 & S3 & S4 & S5 --> R{"fail ≥2 of 5?"}
    R -->|"no"| L["PP-DocLayout pass<br/>tables/formulas? → Tier 2<br/>else → Tier 1"]
    R -->|"yes"| G["Tier 3 (VLM)"]
```

## Pluggable slots

Every tier implements one interface and emits one schema:

```python
class Parser(Protocol):
    def parse(self, page: PageInput) -> list[Block]: ...

@dataclass
class Block:
    text: str
    bbox: tuple[float, float, float, float]
    type: BlockType          # text | table | formula | figure | ...
    confidence: float
    source_tier: int         # 1 | 2 | 3
    page_index: int
```

Tier 3 hides behind an OpenAI-compatible vLLM endpoint, so swapping the VLM is a config change (URL + model name), not a code change. The router is a policy object reading thresholds from config.

| Slot | Pick | License | Swap candidates |
|---|---|---|---|
| PDF signals / split | pypdfium2 | BSD/Apache | pdfplumber (MIT) |
| Layout signal | PP-DocLayout | Apache-2.0 | — (DocLayout-YOLO is AGPL, avoid) |
| Tier 1 | liteparse | Apache-2.0 | pdfplumber-based extractor |
| Tier 2 | Docling + TableFormer + Tesseract `hun` | MIT / Apache | RapidOCR engine in Docling |
| Tier 3 | PaddleOCR-VL via vLLM | Apache-2.0 | dots.ocr, Granite-Docling (both Apache) |
| Schema | DoclingDocument | MIT | custom Pydantic schema |
| Queue / serving | Celery or RQ + vLLM | BSD / Apache | docling-serve, Ray |

### Serving the VLM tier

```bash
vllm serve PaddlePaddle/PaddleOCR-VL --trust-remote-code \
  --max-num-batched-tokens 16384
# clients POST page images to /v1/chat/completions
```

## Excluded for licensing

| Excluded | Reason | Replacement |
|---|---|---|
| PyMuPDF / pymupdf4llm | AGPL-3.0 | pypdfium2 + liteparse |
| Marker | GPL-3.0 code + OpenRAIL-M weights | Docling (tier 2), PaddleOCR-VL (tier 3) |
| Surya (weights), DocLayout-YOLO | OpenRAIL-M / AGPL | PP-DocLayout |
| olmOCR 2 | Apache but English-focused | PaddleOCR-VL (multilingual) |
| MinerU | Permissive-ish 2026 license, but attribution conditions | Optional — add only if its hybrid mode wins on your corpus |

## Hungarian specifics

1. **Tier 3 model choice is forced.** olmOCR 2 is English-focused, so it's out. PaddleOCR-VL's 109-language coverage includes Hungarian; dots.ocr covers it too. Benchmark both on *your* scans — published multilingual scores rarely break out Hungarian.
2. **The double acute is your canary.** The classic Hungarian OCR failure is ő/ű collapsing into ö/ü or õ/û. Build a small eval set of pages dense in ő/ű and make double-acute accuracy an explicit per-tier metric. Tesseract `hun` handles it well on clean scans; VLM output on degraded scans is where drift appears.
3. **Validation must be Hungarian-aware.** The dictionary-word-ratio garble check needs the hu_HU hunspell dictionary (hunspell lib is MPL-optioned tri-license; dictionary files as runtime data don't affect your code's license). An English wordlist would flag every correct Hungarian page as garbage and escalate the whole corpus to the GPU.
4. **Legacy encodings route to OCR.** Set `lang=hun` explicitly in Tesseract (no autodetect), and add a mojibake pattern for Latin-2 / CP-1250 text layers — older Hungarian PDFs often carry broken `õ`-for-`ő` embedded text, which should skip tier 1 entirely.

## Rollout plan

| Stage | Scope |
|---|---|
| 1 | Router + tier 1: pypdfium2 signals, "fail 2 of 5" voting rule, liteparse, DoclingDocument normalization with `source_tier` from day one |
| 2 | Tier 2 + validation: PP-DocLayout signal, Docling+TableFormer+Tesseract `hun`, hu_HU garble checks, escalation logging |
| 3 | Tier 3: PaddleOCR-VL on vLLM, async batched page images, temperature-ramp retries, document-level max error rate (~0.004) |
| 4 | Cross-page table/paragraph merge, page-image hash cache, per-tier cost and escalation-rate metrics |

Primary tuning signal: the escalation matrix (tier 1 → 3 rate). Escalation >~30% means the router thresholds are miscalibrated or the corpus is scan-heavier than assumed.
