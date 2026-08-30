# Log

Chronological record of all operations.

## [2026-08-27] setup | Vault initialized
Created vault "second-brain" for a knowledge base on document intelligence — OCR, layout analysis, document parsing, and information extraction.
Agent configs: CLAUDE.md, AGENTS.md, .cursor/rules/second-brain.mdc.

## [2026-08-27] setup | Vault folder renamed
Renamed the vault folder from `second-brain` to `document-intelligence`. Updated the vault-name references in CLAUDE.md, AGENTS.md, and .cursor/rules/second-brain.mdc. The `second-brain` skill family (/second-brain-ingest, -query, -lint) is unchanged.

## [2026-08-27] ingest | 14 sources — the OCR/VDU and agentic-RAG research corpus
Processed all 14 files in raw/ in date order ascending (2026-08-24 19:51 through 2026-08-26 21:44).
Created 116 pages: 14 source summaries, 59 entities, 43 concepts. Zero orphan wikilinks.
Approach note: sources were read in date order and the wiki written to reflect the full chronological
accumulation, with later corrections recorded as corrections rather than rewriting earlier pages 10 times.

Source files processed, in date order:
1. Open-Source OCR per Visual Document Understanding Models for Self-Hosted AKS Pipelines - A Comparative Investigation (August 2026).md
2. ocr-serving-recipes-runpod-v2.md
3. ocr-api-call-strategy.md
4. ocr-rasterize-encoding-bottlenecks.md
5. ocr-language-models-deployment-critical-pass-2.md
6. ocr-vdu-complete-study.md
7. ocr-arxiv-github-technical-review.md
8. Deep Technical Review - Open-Source OCR - Visual Document Understanding Models - Missed Details and Pipeline Implications.md
9. Local PDF-Markdown and Document Parsing - State of the Art (August 2026).md
10. tiered-pdf-pipeline-architecture.md
11. SOTA Agentic RAG Reference Architecture - A System-Design Session Brief.md
12. Query Modification for a SOTA Agentic RAG System - Taxonomy Benchmarks and a Staged Design.md
13. The Everything Else Map of SOTA RAG Optimization (2024–2026) - A Decision-Ready Survey for a Greenfield Agentic RAG System.md
14. Deep-Dive - 10 RAG Optimization Techniques for a Self-Hosted Agentic Stack.md

Contradictions recorded across sources (see the relevant pages):
- PaddleOCR-VL fine-tuning: rated "less turnkey" in source 5, documented as officially UNSUPPORTED in sources 7 and 8.
- Hungarian in PaddleOCR-VL: "probable, unverified" in sources 5 and 6; VERIFIED in sources 7 and 8.
- GLM-OCR language count: "100+ languages" marketing vs the paper's footnote-2 list of exactly 8.
- Qwen3-VL OCR languages: README "32 up from 10" vs some HF cards "up from 19" vs the paper's 39 (32 above 70%).
- BGE-M3 as primary embedder: recommended in sources 11 and 12, called outdated versus Qwen3-Embedding in 13 and 14.

## [2026-08-27] lint | Health check

Found 3 errors, 2 warnings, 2 info items. All fixed.

**Errors fixed**
1. **Wikilinks did not resolve in Obsidian.** Filenames are kebab-case, links are Title Case, and no page declared an alias — so essentially every internal link was dead. Added `aliases: ["<H1 title>"]` to all 116 pages. Root cause was a conflict inside the wiki schema itself (kebab-case filenames + Title Case links), so the alias rule was also written into `CLAUDE.md`, `AGENTS.md` and `.cursor/rules/second-brain.mdc` to stop the next ingest reintroducing it.
2. **6 broken links from index.md to source pages** — source filenames drop stop words their titles keep. Resolved by the same alias fix.
3. **Units error in `entities/olmocr-bench.md`** — liteparse listed as `0.391` in a column of percentages. Corrected to `39.1`, row re-sorted into descending order, and `MinerU` linked in the same table.

**Warnings fixed**
4. **Five load-bearing unverified claims checked against primary sources.** Results: Hungarian is **NOT** among `opensearch-neural-sparse-encoding-multilingual-v1`'s 15 languages (`bn te es fr id hi ru ar zh fa ja fi sw ko en`); Hungarian is **NOT** declared by LightOnOCR (`LightOnOCR-2-1B` card lists 11: `en fr de es it nl pt sv da zh ja`, no CEE language); the GLM-OCR CogViT connector downsamples **4x** (`spatial_merge_size: 2`, `patch_size: 14`, so one visual token = 28x28 px) — read from `config.json`, which the paper never states; Qwen3-VL's "32 languages" in the docs is the **>70%-accuracy count**, not the support count (39 supported), which retires that contradiction. Still open: Hungarian membership in Qwen3-VL's roster, which exists only as Figure 2, a raster image. GLM-OCR MTP-after-LoRA-merge remains vendor-undocumented, but narrowed to a one-hour measurement (draft acceptance rate from vLLM `/metrics`) with a documented mitigation (gated LoRA, arXiv 2507.11851).
5. **`wiki/synthesis/` was empty.** Created 4 pages: [[Hungarian Model Decision Matrix]], [[The OCR-to-RAG Seam]], [[Cost per Page Model]], [[Open Inputs and Corpus Profile]].

**Info fixed**
6. **12 unlinked entity mentions** linked across 9 pages (Neo4j, ColPali, BGE-M3, Qwen3-Embedding, Unsloth, MinerU, and vLLM in 6 places).
7. **Data gaps** consolidated into [[Open Inputs and Corpus Profile]], separated into "only you can answer" (volume, born-digital/scanned split, page-length distribution, latency class, data residency, plus the golden set) versus "externally answerable".

**Not fixed, deliberately:** the eval-harness implementation gap (CER/TableTEDS libraries, the diacritic metric, `hu_HU` hunspell wiring). Closing it means writing content no ingested source contains, so it stays flagged in [[Open Inputs and Corpus Profile]] rather than invented.

**Post-fix state:** 120 pages, 120 aliases, 120 distinct wikilinks, 0 broken links, 0 orphans, 0 pages missing from the index.

**New pages:** [[Hungarian Model Decision Matrix]], [[The OCR-to-RAG Seam]], [[Cost per Page Model]], [[Open Inputs and Corpus Profile]].

## [2026-08-27] query | How to query this wiki

Asked what questions the accumulated corpus can usefully answer. Mapped the 120 pages into seven
question types that get a strong answer (decisions, trade-offs, failure modes, cost/sizing,
architecture opinions, licensing/sovereignty, recorded contradictions), and separated them from the
zone the wiki structurally cannot answer — the five corpus facts in [[Open Inputs and Corpus Profile]].

Saved as [[How to Query This Wiki]]. Index updated under Synthesis.

**New pages:** [[How to Query This Wiki]].

## [2026-08-27] verify | Qwen3-VL Figure 2 roster, and the Qwen3.5-3.8 generations

Closed the wiki's highest information-per-minute open item, and then re-verified the whole Qwen
generation map against primary sources. Both results changed load-bearing claims.

**1. Figure 2 read (arXiv 2511.21631, page 17).** Downloaded the PDF, extracted the embedded raster
(4769x2669, no text layer) and rendered the axis labels at 600 dpi. The figure plots **exactly 32
bars, not 39** -- it shows only the languages above the 70% threshold. **Hungarian is not among
them**, the 7 supported languages below the threshold are named nowhere in the paper, and the string
"Hungarian" does not occur in the PDF at all. Either branch eliminates Qwen3-VL as a zero-shot
Hungarian arm. Full roster recorded in [[Hungarian OCR and the Double Acute Test]]. Noted as
inference, not as a paper claim: the weakest languages on the chart are Romanian (~71) and Polish
(~74) while Swedish/Danish reach 97-98, i.e. dense-diacritic Latin scripts cluster at the bottom.

**2. Generation map corrected.** The previous table had two wrong rows. Verified against model cards
and repos: Qwen3.7 **never shipped open weights** (a skipped generation, API-only); Qwen3.8 exists
(27B + 2.4T-A95B, Apache-2.0, `image-text-to-text`, OmniDocBench 1.5 = 91.1). Qwen3.6 is natively
multimodal with OCR, contradicting the earlier "zero OCR evidence" note -- but it starts at 27B.

**3. The "-VL suffix" question settled with numbers.** From Qwen's own Qwen3.5-9B card, one harness:
Qwen3.5-4B matches Qwen3-VL-30B on document parsing (OmniDocBench 1.5 86.2 vs 86.8, CC-OCR 76.7 vs
77.8) and 9B beats it on every row (OCRBench 89.2 vs 83.9). CharXiv (RQ) jumps +16.4 points, which
is the benchmark [[The OCR-to-Text Boundary Limit]] is built on -- so that page now carries a note
that the cheapest route around the boundary may be an end-to-end model rather than [[ColPali]].

**4. Two corrections to the recommendation.** The ladder default moved from **Qwen3.5-4B to 9B**: a
practitioner report finds 0.8B-4B drift into summarising documents instead of transcribing them --
the instruction-side twin of [[Linguistic Crutch and Faithfulness]], and invisible to a CER-only
check. And the size ladder **closes after 3.5**: 0.8B-9B open weights exist only in that generation,
so 3.5 is the terminus for this slot, not a waypoint.

**New open item:** Qwen3-VL documents QwenVL-HTML with `data-bbox`; the 3.5/3.6/3.8 repos document
no equivalent. Two of the four fields [[The OCR-to-RAG Seam]] requires depend on it. Recorded in
[[Open Inputs and Corpus Profile]] as the replacement for the closed Figure 2 item.

**Pages updated:** [[Qwen Model Family]] (rewritten), [[Hungarian OCR and the Double Acute Test]],
[[Hungarian Model Decision Matrix]], [[Open Inputs and Corpus Profile]], [[The OCR-to-RAG Seam]],
[[The OCR-to-Text Boundary Limit]], [[How to Query This Wiki]], index.

**No new pages.**

## [2026-08-27] query | DeepSeek-OCR-2 on the matrix, and the SDK-as-harness axis

Two gaps found by the user, both real.

**1. DeepSeek-OCR-2 was in the wiki but not on the matrix.** The entity page has carried OCR-2 since
ingest (arXiv 2601.20552, DeepEncoder V2, Visual Causal Flow), and both [[olmOCR-Bench]] and
[[Multi-Script OCR Benchmarks]] score it separately from v1 -- but [[Hungarian Model Decision Matrix]]
had a single `DeepSeek-OCR` row that **mixed v1 evidence with v2 as the candidate**. Split into two
rows and verified against the model card: OCR-2 is Apache-2.0 repo (same AGPL PyMuPDF toolchain
wrinkle), olmOCR-Bench 76.3, OmniDocBench overall 73.01, 35 published finetunes -- **and still
socOCRbench ~0.176 against dots.ocr ~0.478**. The split verdict is the point: v2 supersedes v1 on
every clean-print number and closes none of the multi-script gap. Probe order updated to say probe
and train on **v2**, keeping v1 only as the historical baseline; the 9.2% catastrophic cohort is a
v1 measurement and must not be carried forward.

**2. The harness was never a row.** Added a section making the [[glmocr SDK]] an explicit axis of the
matrix: every model row is a config value behind the same pipeline, the three swap options (name
alias / adapter proxy / subclass) with their verdicts, and two consequences -- that [[GLM-OCR]] the
model being disqualified says nothing about the SDK, and that a model unable to emit `bbox` is not
disqualified either, because [[PP-DocLayout-V3]] supplies coordinates independently. That last point
is the fallback for the open Qwen3.5 `data-bbox` question.

**Probe order gained a step 0:** stand up `--served-model-name glm-ocr` against any vLLM service
before evaluating anything, so each candidate is a one-variable swap instead of an integration. The
SDK page's Option-A example was aligned from 4B to 9B to match the revised ladder.

**New benchmark recorded:** MDPBench (arXiv 2603.28130) -- 3,400 images, 17 languages, real-world;
open models drop 17.8% on photographed documents and 14.0% on non-Latin scripts. Hungarian membership
unconfirmed and added to [[Open Inputs and Corpus Profile]] as a cheap external check.

**Pages updated:** [[Hungarian Model Decision Matrix]], [[DeepSeek-OCR]], [[glmocr SDK]],
[[Multi-Script OCR Benchmarks]], [[Open Inputs and Corpus Profile]], index.

**No new pages.**

## [2026-08-27] verify | ColPali as an architecture, and versions on the matrix

**1. ColPali was under-specified as a single model.** Rewritten around the fact that it is *late
interaction applied to a backbone* -- the backbone is a config value, the same inversion as
[[Pipeline as Platform, Model as Config]] on the OCR side. Verified the full checkpoint table against
the upstream `illuin-tech/colpali` README and the `vidore` org page on 2026-08-27:

- `vidore/colpali` v1.0-v1.3 (PaliGemma-3B, **Gemma licence**, 81.3-84.8) -- **fails the permissive
  rule outright**, which the page had recorded but not made decisive.
- `vidore/colqwen2` v0.1-v1.0 (Qwen2-VL-2B, Apache-2.0, 87.3-89.3); `vidore/colqwen2.5` v0.1-v0.2
  (Qwen2.5-VL-3B, 88.8-89.4); ColSmol 256M/500M (80.1/82.3); ModernVBERT v1.0.
- **The two best checkpoints are community repos, not vidore:** `TomoroAI/tomoro-colqwen3-embed-4b`
  (Qwen3-VL, 90.6) and `athrael-soju/colqwen3.5-4.5B-v3` (**Qwen3.5-4B**, 90.9).

Three findings changed the page's advice. **`colpali-engine` is deprecated** -- the README now
recommends Sentence Transformers for new projects, so the old "embeddings via colpali-engine" serving
note was already wrong. **A live licence conflict** exists on colqwen2.5-v0.2: the README says
Apache-2.0, the model card says Qwen RESEARCH LICENSE; both read the same day, recorded rather than
resolved. And **no ColPali variant declares Hungarian** -- `tsystems/colqwen2.5-3b-multilingual`
covers 5 languages, Hungarian not among them -- but the failure mode moves from diacritic recognition
to cross-lingual query alignment, which is a different and likely easier problem. Recorded honestly
as untested, not as proven.

**Backbone convergence noted:** the top ColPali checkpoint and the recommended recognition arm both
run on Qwen3.5-4B/9B. One family could serve both slots -- one serving pattern, one VRAM profile
family, one Gated-DeltaNet caveat. Added to [[Hungarian Model Decision Matrix]] as "the route that
skips this table entirely", alongside a four-axis comparison against the recognition slot.

**2. Versions added to the matrix.** Every row now carries an explicit version column: PaddleOCR-VL
**1.6** (with its OmniDocBench figures flagged as 1.5), dots.ocr **1.5**, HunyuanOCR **1.5**, MinerU
**2.5**, olmOCR **2**, LightOnOCR **2-1B**, DeepSeek-OCR **v1** and **OCR-2** with arXiv ids, Qwen3-VL
(Dec 2025) and Qwen3.5 (Feb 2026). [[GLM-OCR]] publishes no version number -- recorded as such rather
than invented. The rationale is on the page: the DeepSeek lineage produced a wrong verdict here
precisely because an undated row mixed two versions.

**Pages updated:** [[ColPali]] (largely rewritten), [[Hungarian Model Decision Matrix]], index.

**No new pages.**

## [2026-08-27] verify | Baidu Unlimited-OCR: the missing row and the missing language roster

The user asked why [[Baidu Unlimited-OCR]] is absent from [[Hungarian Model Decision Matrix]]. It was
absent because it is a whole-document model rather than a per-page one, and the matrix is implicitly
scoped to the per-page recognition slot -- but that scoping was never written down, so the absence
read as an oversight. It is MIT and holds the highest OmniDocBench score anywhere in this wiki
(93.23 v1.5 / 93.92 v1.6). Added, with the shape difference stated rather than assumed.

**Checked against the model card and repo on 2026-08-27:** there is **no language roster**. The card
carries a bare `multilingual` tag, enumerates nothing, never mentions Hungarian, and publishes no
multilingual benchmark -- not socOCRbench, not GlotOCR, not MDPBench, not OmniDocBench multilingual.
The only leaderboard figure on the card is ParseBench (mean 46.17). Community reports cover Cyrillic
and handwritten maths; nothing on Central European diacritics. New facts recorded: 3.3B params, and
**ms-swift training support**, which the entity page did not have.

**The structural finding, and the reason this is more than a missing cell.** The model is
continue-trained from [[DeepSeek-OCR]] with the **DeepEncoder frozen** -- and that freeze is not
incidental, it is *why* [[R-SWA Reference Sliding Window Attention]] works at all, since reference
tokens must stay outside the recursive state. The same choice means the model cannot have gained
visual script competence its base lacked, and DeepSeek-OCR v1 is the weakest multi-script model in
the field. So long-horizon capability was bought on top of a weak multilingual foundation, by a
mechanism that forecloses fixing that foundation through continued pre-training. Adapting it to
Hungarian is decoder-side work with an inherited encoder ceiling -- the same work as adapting
DeepSeek-OCR.

Added as **structural fact 4** on the matrix: the best benchmark score in the field comes with the
least language evidence, which is the cleanest example of [[Benchmark Saturation]] on the table.
Probe order gained it at position 5, explicitly as a *fidelity* arm gated on the page-length
distribution -- an [[Open Inputs and Corpus Profile]] input -- and explicitly not pulled forward by
its OmniDocBench score.

**Pages updated:** [[Baidu Unlimited-OCR]], [[Hungarian Model Decision Matrix]], index.

**No new pages.**

## [2026-08-27] lint | Full coverage audit — every named model given a dated verdict

The user asked for a systematic pass rather than model-by-model questions: find what is missing, add
it, and justify every exclusion with evidence. Inventoried all 59 entity pages and every benchmark
table against the [[Hungarian Model Decision Matrix]], then verified each gap against model cards,
repository READMEs and papers.

**New page: [[Coverage and Exclusion Register]].** Every model this wiki names now has an explicit
verdict with a source and a date. The rule it enforces: a model absent from the matrix must be
*excluded on stated evidence*, never merely absent, because that is precisely how DeepSeek-OCR-2,
ColPali's backbones and Baidu Unlimited-OCR each went missing earlier this week.

**Two models were absent from the wiki entirely; both added.**
- [[OvisOCR2]] (arXiv 2607.13639, Jul 2026) -- Apache-2.0, **0.8B on a Qwen3.5-0.8B backbone**,
  self-reporting OmniDocBench v1.6 96.58 and PureDocBench 75.06. No declared languages.
- [[Qwen3-VL-Embedding]] / Qwen3-VL-Reranker (arXiv 2601.04720) -- 2B/8B unified multimodal
  retrieval, MMEB-V2 77.8 (first at the paper's cutoff), 30+ languages, MRL, 32k context. It is the
  **single-vector alternative to [[ColPali]]**, and the RAG half had no page for it, which had made
  visual retrieval look like an architecture question with one answer.

**Four corrections where a fresh read overrode an ingested claim.**
1. [[Marker and Chandra]]: the weights threshold is **$5M funding/revenue**, not the $2M on the page
   -- `datalab-to/marker` README, quoted verbatim.
2. **Nanonets**: the wiki treated OCR-3 as a ~3-4B Apache-2.0 model leading olmOCR-Bench at 87.4.
   **There is no open OCR-3.** `nanonets/` publishes OCR-s, OCR2-3B and OCR2-1.5B-exp only; OCR2-Plus
   is Docstrange API. The open OCR2-3B scores **69.5**, names 11 languages, and Hungarian is not one.
3. **Nemotron Parse 2.0**: OpenMDW-1.1 plus the NVIDIA Open Model License -- not standard permissive
   -- and its multilingual expansion targets **Indic scripts** (IndicVisionBench, MOSCAR).
4. [[Granite-Docling]]: **English primary**, with Japanese/Arabic/Chinese experimental and no
   Hungarian; the card also states it is "not intended for general image understanding".

**One structural defect fixed.** [[olmOCR-Bench]] ranked downloadable weights and hosted products in
a single column, so the leaderboard read as if its top entries were self-hostable. An availability
column now marks each row: of the four systems above 83, one is hosted-only and two carry a
revenue-capped weights licence. Under the permissive-only rule the effective open leader there is
[[LightOnOCR]]-2-1B at 83.2 -- which declares no Hungarian.

**A grade-of-evidence rule now applies to the matrix.** Read from `opendatalab/OmniDocBench` on
2026-08-27 (v1.6, updated 2026-04-10): PaddleOCR-VL-1.6 **96.34**, MinerU2.5-Pro 95.75, GLM-OCR
95.22, dots.ocr 90.77, DeepSeek-OCR 2 90.25. [[Baidu Unlimited-OCR]] and [[OvisOCR2]] appear on **no
official leaderboard** -- their 93.92 and 96.58 are self-reports. Where a self-report outranks the
verified leader by less than the benchmark's own noise band, the verdict is *unproven*, not *better*.
This reordered two rows: [[PaddleOCR-VL]] is the leaderboard leader and was previously described only
as "strong on messy scans", and [[GLM-OCR]] is 3rd at 95.22 while sitting at OCR Arena #24 -- recorded
as a split verdict rather than "weak".

**New open item:** PaddleOCR-VL's Hungarian verification was done against an earlier release's
language list while every other cell in its row is now 1.6. It is the only verified-Hungarian row on
the matrix, so re-checking it against 1.6 is worth more than it costs. Added to
[[Open Inputs and Corpus Profile]].

**Explicit non-verdicts, recorded rather than hidden:** MonkeyOCR-Pro, FireRed-OCR, Youtu-Parsing and
InternVL are named by the corpus with no distinguishing evidence. They are neither included nor ruled
out, and the register says so.

**New pages:** [[Coverage and Exclusion Register]], [[OvisOCR2]], [[Qwen3-VL-Embedding]].

**Pages updated:** [[Hungarian Model Decision Matrix]], [[olmOCR-Bench]], [[OmniDocBench]],
[[Marker and Chandra]], [[Other OCR Contenders]], [[Open Inputs and Corpus Profile]], index.

## [2026-08-27] verify | PaddleOCR-VL Hungarian: confirmed as membership, not as accuracy

Re-checked the only verified-Hungarian row on [[Hungarian Model Decision Matrix]], from the primary
PDFs rather than from cards or search.

**Confirmed.** arXiv 2510.14528, Appendix B "Supported Languages", Table A1 names Hungarian in the
Latin category, verbatim: "...Croatian, Uzbek, Hungarian, Serbian (Latin)...". It is a **text table,
not a raster image**, so this is a hard read. The Latin group holds **47 languages** of the model's
109. The wiki's existing claim was accurate.

**Three things the re-check added, and the third changes the verdict's meaning.**

1. **The roster belongs to v1 and was never restated.** The 1.6 technical report (arXiv 2606.03264,
   Jun 2026) is a region-refinement and RL post-training paper and **does not mention languages at
   all**; the 1.6 model card enumerates none either. So 1.5 and 1.6 *inherit* the roster rather than
   reconfirm it, and RL post-training can in principle shift multilingual behaviour. No evidence
   either way -- recorded as a reasonable but unverified inheritance.

2. **The measurement is vendor data** -- Baidu's self-built In-house-OCR set, 107,452 line-level
   samples, not an independent benchmark.

3. **There is no Hungarian-specific accuracy number anywhere.** The paper points at Table 6a for
   per-language performance, but **Table 6a scores by script group, not by language**: PaddleOCR-VL
   reads **Latin = 0.013 edit distance**, best of all ten groups -- across all 47 Latin languages
   from French to Quechua. **A Latin-group mean dominated by French, German and Spanish is exactly
   where a Hungarian double-acute failure would be invisible.**

**Net:** the row keeps its checkmark, but the checkmark now says what it actually means -- *Hungarian
is a declared, trained language*, not *Hungarian accuracy is measured*. The open item this closes is
replaced by a sharper one: the question is no longer "is Hungarian supported" but "what is the o-double-acute
and u-double-acute error rate", and only [[Golden Set and Eval Harness]] answers that.

**Pages updated:** [[PaddleOCR-VL]], [[Hungarian Model Decision Matrix]],
[[Hungarian OCR and the Double Acute Test]], [[Open Inputs and Corpus Profile]].

**No new pages.**
