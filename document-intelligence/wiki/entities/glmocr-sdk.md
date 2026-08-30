---
aliases: ["glmocr SDK"]
tags: [pipeline, sdk, architecture, tool]
sources: [ocr-language-models-deployment-critical-pass-2.md, ocr-vdu-complete-study.md, ocr-arxiv-github-technical-review.md, Deep Technical Review - Open-Source OCR - Visual Document Understanding Models - Missed Details and Pipeline Implications.md]
created: 2026-08-27
updated: 2026-08-27
---

# glmocr SDK

The Python pipeline shipped with [[GLM-OCR]] (`pip install "glmocr[selfhosted]"`). Critically, it is **kept even after the model it was built for is rejected** — it is the concrete instance of [[Pipeline as Platform, Model as Config]].

## What it contains (the durable value)

[[PP-DocLayout-V3]] integration · parallel region fan-out · reading-order assembly · JSON/Markdown formatting via `ResultFormatter` · a Flask server (`python -m glmocr.server`, port 5002, endpoint `/glmocr/parse`, a list of images = pages of ONE document) · `--layout-device cpu|cuda:N` placement. Exported modular classes: `PageLoader`, `OCRClient`, `PPDocLayoutDetector`, `ResultFormatter`.

`OCRClient` calls a plain OpenAI-compatible endpoint, which is what makes the recognition model a config value rather than a code dependency.

## The env-var override discovery

The SDK maps dotted config paths to `GLMOCR_`-prefixed environment variables:

```
pipeline.ocr_api.api_host  →  GLMOCR_PIPELINE_OCR_API_API_HOST
```

Repointing the SDK at any [[vLLM]] service therefore requires **zero code and zero YAML edits** — a pure Kubernetes ConfigMap change. Config priority chain: constructor kwargs > `os.environ` > `.env` > `config.yaml` > defaults. Other keys: `pipeline.maas.enabled`, `pipeline.ocr_api.api_port` (default 8080), `page_loader.min_pixels: 12544 / max_pixels: 71372800`, `result_formatter.output_format: both`, `layout.device`. Region bboxes use a normalized 0–1000 scale.

## The three swap options

| Option | Shape | Verdict |
|---|---|---|
| **A — name alias** | `vllm serve Qwen/Qwen3.5-9B --served-model-name glm-ocr` | Runs end-to-end immediately, but quality degrades: GLM-OCR's fixed task prompts meet a different model's output dialect (preambles, code fences, QwenVL-HTML) and `ResultFormatter` mangles it. **The smoke test and probe vehicle, not production.** |
| **B — adapter proxy** | ~200-line shim speaking "GLM dialect" to the SDK and engineered per-region-type prompts to the model, normalizing responses back | **Recommended production shape.** The SDK stays a pinned, unmodified pip dependency; the adapter is the anti-corruption layer where every future model slots in. |
| **C — subclass `OCRClient`** | in-process | The package is explicitly composable, but this couples you to young internal interfaces. Only if B's network hop measurably hurts — at region granularity it will not. |

**End state: B plus a format-native fine-tune**, at which point the adapter's normalization shrinks toward identity. See [[LoRA Fine-Tuning for OCR]].

**Option A is also the cheapest probe harness in the project.** One `--served-model-name glm-ocr` flag turns any [[vLLM]] service into a running end-to-end pipeline, which makes every candidate on the [[Hungarian Model Decision Matrix]] a one-variable swap rather than an integration. It is listed there as step 0 of the probe order — run before any model is evaluated, precisely because its output quality does not matter yet.

## CI guardrails

The SDK, the adapter and the weights now version independently, and any of the three can silently break the output contract. Three region-level fixtures run against the live pair guard the whole surface:

1. a **table** region → parseable HTML
2. a **formula** region → bare LaTeX
3. a **text** region containing **ő/ű**

## Caution

The SDK is young. Pin its version and expect the env-var convention and internal interfaces to churn — the Option-B adapter shim is the insulation. The exact fixed prompt strings live in the package source, not the paper; extract them from source for the adapter.

## Related

[[GLM-OCR]] · [[Qwen Model Family]] · [[neural-maze production-ocr-course]] · [[Pipeline as Platform, Model as Config]] · [[PP-DocLayout-V3]] · [[Hungarian Model Decision Matrix]]
