# Serving Recipes: OCR/VDU Models on vLLM & SGLang + RunPod API v2 Templates
*Compiled August 2026. Each model has: (a) minimal-GPU config, (b) H100-80GB optimized config, (c) RunPod template payload. Verify flags against each repo before production — these stacks move monthly.*

---

## GPU sizing at a glance

| Model | Weights (BF16) | Practical minimum GPU | Comfortable | H100 fit |
|---|---|---|---|---|
| GLM-OCR (0.9B) | ~2 GB | RTX 3060 12 GB / T4 16 GB* | L4 / A10 24 GB | Trivial — pack replicas or MIG |
| PaddleOCR-VL-1.6 (0.9B) | ~2 GB (pipeline ~11–13 GB via vLLM) | **Ampere+ required (CC ≥ 8.0)** → A10/L4 | L4 / A10 | Easy, high batch |
| DeepSeek-OCR / OCR-2 (3B MoE) | ~6.7 GB | RTX 4090 / L4 16–24 GB | A10 / L40S | Easy |
| Unlimited-OCR (3B MoE) | ~6.7 GB | RTX 4090 / L4 (transformers path) | A100 | **Natural home — SGLang FA3 path is Hopper-only** |
| dots.ocr (1.7B) | ~4 GB (runtime heavy) | L4 / A10 24 GB (tight) | A100 40 GB | Easy |
| MinerU 2.5 (1.2B VLM backend) | ~3 GB | 8 GB VRAM (Volta+); pipeline backend: 4 GB / CPU | L4 / A10 | Easy |

\* GLM-OCR itself runs on old GPUs, but vLLM ≥ 0.19 effectively wants Ampere+ for the fast paths; on Turing use transformers/Ollama fallback.

All models below expose an OpenAI-compatible endpoint on the given port.

---

## 1. GLM-OCR (`zai-org/GLM-OCR`)

Prereqs: `transformers>=5.3.0`, vLLM ≥ 0.19 (`vllm/vllm-openai:v0.19.0-ubuntu2404`) or SGLang ≥ 0.5.10 (`lmsysorg/sglang:v0.5.10`).
⚠️ For paper-level accuracy, front the endpoint with the `glmocr` SDK pipeline (`pip install "glmocr[selfhosted]"`) — it runs PP-DocLayout-V3 layout + parallel region OCR against your server (`pipeline.ocr_api.api_host/port` in `config.yaml`). Serving the bare VLM skips layout analysis.

### vLLM — minimal (12–16 GB GPU)
```bash
vllm serve zai-org/GLM-OCR \
  --port 8080 \
  --served-model-name glm-ocr \
  --speculative-config '{"method": "mtp", "num_speculative_tokens": 3}' \
  --max-model-len 8192 \
  --gpu-memory-utilization 0.85 \
  --max-num-seqs 16
```

### vLLM — H100 optimized
```bash
vllm serve zai-org/GLM-OCR \
  --port 8080 \
  --served-model-name glm-ocr \
  --speculative-config '{"method": "mtp", "num_speculative_tokens": 3}' \
  --max-model-len 32768 \
  --gpu-memory-utilization 0.92 \
  --max-num-seqs 256 \
  --no-enable-prefix-caching
```
(MTP speculative decoding is the big win — ~5 tokens/step. Prefix caching off: OCR requests share no prefix, the cache only wastes memory. On H100 consider 2–4 replicas via MIG instead of one giant instance — the 0.9B model saturates on preprocessing before it saturates the GPU.)

### SGLang — minimal
```bash
SGLANG_ENABLE_SPEC_V2=1 sglang serve \
  --model-path zai-org/GLM-OCR \
  --port 8080 \
  --served-model-name glm-ocr \
  --speculative-algorithm NEXTN \
  --speculative-num-steps 3 --speculative-eagle-topk 1 --speculative-num-draft-tokens 4 \
  --mem-fraction-static 0.8 --context-len 8192
```

### SGLang — H100
Same command with `--context-len 32768 --mem-fraction-static 0.9` and raise `--max-running-requests`.

---

## 2. PaddleOCR-VL-1.6 (`PaddlePaddle/PaddleOCR-VL-1.6`)

⚠️ Hard requirement: NVIDIA compute capability ≥ 8.0 (Ampere+). T4/V100 are known to OOM/timeout. Like GLM-OCR, accuracy requires the **full two-stage pipeline** — serve the VLM, then run the PaddleOCR CLI/SDK against it (`paddleocr doc_parser ... --vl_rec_backend vllm-server --vl_rec_server_url http://HOST:8080/v1`). FastDeploy is Baidu's fastest backend (~2.0 pages/s on A100) but vLLM is the more portable choice.

### vLLM — minimal (A10/L4 24 GB)
```bash
vllm serve PaddlePaddle/PaddleOCR-VL-1.6 \
  --port 8080 \
  --trust-remote-code \
  --served-model-name paddleocr-vl \
  --max-model-len 16384 \
  --gpu-memory-utilization 0.6 \
  --max-num-seqs 32
```
(Low `gpu-memory-utilization` deliberately — leave headroom for the layout stage if co-located; move layout to CPU pods in your event-driven setup instead.)

### vLLM — H100
```bash
vllm serve PaddlePaddle/PaddleOCR-VL-1.6 \
  --port 8080 --trust-remote-code --served-model-name paddleocr-vl \
  --max-model-len 16384 \
  --gpu-memory-utilization 0.9 \
  --max-num-seqs 256 \
  --no-enable-prefix-caching
```
The bottleneck on H100 becomes the CPU-side PP-DocLayoutV3 stage — scale layout workers horizontally, not the GPU.

### SGLang — minimal / H100
```bash
python -m sglang.launch_server \
  --model-path PaddlePaddle/PaddleOCR-VL-1.6 \
  --trust-remote-code \
  --port 8080 \
  --mem-fraction-static 0.7 \
  --context-length 16384
# H100: --mem-fraction-static 0.9 and raise concurrency
```

---

## 3. DeepSeek-OCR / DeepSeek-OCR-2 (`deepseek-ai/DeepSeek-OCR[-2]`)

⚠️ The n-gram logits processor is **not optional** — it is the repetition-loop guardrail. Prefix caching and the multimodal processor cache must be disabled. OCR-2 needed newer vLLM than v1; use a recent build.

### vLLM — minimal (16 GB GPU)
```bash
vllm serve deepseek-ai/DeepSeek-OCR \
  --port 8080 \
  --trust-remote-code \
  --served-model-name deepseek-ocr \
  --logits-processors vllm.model_executor.models.deepseek_ocr:NGramPerReqLogitsProcessor \
  --no-enable-prefix-caching \
  --mm-processor-cache-gb 0 \
  --gpu-memory-utilization 0.85 \
  --max-num-seqs 16
```
Per request (OpenAI client `extra_body`):
```json
{"skip_special_tokens": false,
 "vllm_xargs": {"ngram_size": 30, "window_size": 90}}
```

### vLLM — H100
Same flags with `--gpu-memory-utilization 0.92 --max-num-seqs 128`. The MoE decoder (~570M active) is cheap; on H100 this model is throughput-bound on image preprocessing — batch aggressively and rasterize upstream at fixed DPI.

### SGLang
DeepSeek-OCR is supported; launch analogous to Unlimited-OCR below minus the FA3 requirement:
```bash
python -m sglang.launch_server \
  --model-path deepseek-ai/DeepSeek-OCR \
  --trust-remote-code --port 8080 \
  --mem-fraction-static 0.8 \
  --enable-custom-logit-processor \
  --disable-overlap-schedule
```
(vLLM is the better-trodden path for this lineage — the official recipe lives at recipes.vllm.ai. Verify SGLang flag names against the current SGLang docs for your version.)

---

## 4. Baidu Unlimited-OCR (`baidu/Unlimited-OCR`)

Two modes: **gundam** (base_size 1024, image_size 640, crop, fast single-image, `window_size=128`) and **base** (image_size 1024, no crop, multi-page/PDF, `window_size=1024`). Prompt must literally begin with `<image>` (e.g. `<image>document parsing.` / `<image>Multi page parsing.`). Output carries `<|ref|>…<|/ref|>` / `<|det|>…<|/det|>` tokens — strip `<|det|>` boxes to get clean markdown.

### vLLM — dedicated image (works on 16–24 GB, best on Hopper)
```bash
docker run --rm --gpus all --network host --ipc host \
  vllm/vllm-openai:unlimited-ocr \
  baidu/Unlimited-OCR \
  --trust-remote-code \
  --logits_processors vllm.model_executor.models.unlimited_ocr:NGramPerReqLogitsProcessor \
  --no-enable-prefix-caching \
  --mm-processor-cache-gb 0
```
Per request:
```json
{"skip_special_tokens": false,
 "vllm_xargs": {"ngram_size": 35, "window_size": 128}}
```
Use `"window_size": 1024` for multi-page/PDF. Minimal-GPU tuning: add `--max-model-len 16384 --gpu-memory-utilization 0.85`; H100: `--max-model-len 32768 --gpu-memory-utilization 0.92 --max-num-seqs 64`.

### SGLang — H100/H200 (FA3 is Hopper-only)
```bash
python -m sglang.launch_server \
  --model baidu/Unlimited-OCR \
  --served-model-name Unlimited-OCR \
  --attention-backend fa3 \
  --page-size 1 \
  --mem-fraction-static 0.8 \
  --context-length 32768 \
  --enable-custom-logit-processor \
  --disable-overlap-schedule \
  --skip-server-warmup \
  --host 0.0.0.0 --port 10000
```
Note: this uses the **dev-build SGLang wheel bundled in the repo** — pin it in your image, don't `pip install sglang` fresh. On non-Hopper GPUs, drop `--attention-backend fa3` and accept a slower long-context path, or use the vLLM route. The R-SWA constant-KV-cache benefit is precisely why this model + H100 + 32K context is the right pairing for long contracts.

---

## 5. dots.ocr (`rednote-hilab/dots.ocr`)

⚠️ Model directory name must contain **no periods** if you download locally (e.g. `DotsOCR`, not `dots.ocr`). Ampere+ needed. Known repetition on runs of `…`/`___` — keep an output-loop detector.

### vLLM — minimal (A10/L4 24 GB, tight)
```bash
vllm serve rednote-hilab/dots.ocr \
  --port 8080 \
  --trust-remote-code \
  --served-model-name dots-ocr \
  --max-model-len 16384 \
  --gpu-memory-utilization 0.9 \
  --max-num-seqs 8
```
(Official vLLM support since v0.11.0. Keep `max-num-seqs` low on 24 GB — this model's runtime memory balloons at high batch.)

### vLLM — H100
Same with `--max-num-seqs 64 --no-enable-prefix-caching`. Even on H100, expect ~0.3–0.4 pages/s per replica — dots.ocr buys accuracy, not speed; scale horizontally.

### SGLang
```bash
python -m sglang.launch_server \
  --model-path rednote-hilab/dots.ocr \
  --trust-remote-code --port 8080 \
  --mem-fraction-static 0.85 --context-length 16384
```
(vLLM is the reference path in the repo; treat SGLang as secondary and validate output parity.)

---

## 6. MinerU 2.5 (platform — brings its own vLLM/SGLang engine)

MinerU wraps the engine, so you deploy MinerU, not raw vLLM:
```bash
pip install "mineru[vllm]"          # or mineru[sglang]

# Server (OpenAI-style + task API), vLLM engine inside:
mineru-openai-server --port 8000    # min GPU: 8 GB (Volta+)

# Batch CLI, engine selection:
mineru -p input.pdf -o ./out -b vlm-vllm-engine     # GPU VLM backend
mineru -p input.pdf -o ./out -b pipeline            # 4 GB / CPU fallback
mineru -p input.pdf -o ./out -b vlm-sglang-engine   # SGLang engine
```
H100: run the vlm engine with its vLLM async server and put `mineru-router` in front for multi-replica load-balancing (>2 pages/s, ~2.3k tokens/s per A100-class replica; more on H100). Exact backend/router flags shift between minor versions — check `mineru --help` on your pinned version.

---

## RunPod REST API v2 — templates for each

API v2 (public beta, 2026) unifies everything under one base URL:
```
https://api.runpod.io/v2          # Bearer auth with your RunPod API key
GET  /v2/openapi.json             # full schema — generate a client from this
POST /v2/templates                # create template
POST /v2/pods                     # launch pod from template
POST /v2/endpoints                # create serverless endpoint from template
```
So yes — one `POST /templates` per model gives you reusable, API-managed configs, and the same template drives both Pods and Serverless workers (`isServerless: true`). If you have an existing v1 (`rest.runpod.io/v1`) or GraphQL integration, RunPod ships a migration path; field names below match the documented template schema (identical in v1 and v2).

### Generic creator script
```bash
create_template () {
  curl -s -X POST https://api.runpod.io/v2/templates \
    -H "Authorization: Bearer $RUNPOD_API_KEY" \
    -H "Content-Type: application/json" \
    -d @"$1"
}
```

### Template payloads

**glm-ocr.json** — smallest/cheapest; pairs with 24 GB GPUs (RTX 4090/L4/A10) or H100:
```json
{
  "name": "glm-ocr-vllm",
  "imageName": "vllm/vllm-openai:v0.19.0-ubuntu2404",
  "category": "NVIDIA",
  "containerDiskInGb": 40,
  "volumeInGb": 20,
  "volumeMountPath": "/root/.cache/huggingface",
  "ports": ["8080/http", "22/tcp"],
  "env": { "HF_HUB_ENABLE_HF_TRANSFER": "1" },
  "dockerStartCmd": [
    "vllm", "serve", "zai-org/GLM-OCR",
    "--port", "8080", "--served-model-name", "glm-ocr",
    "--speculative-config", "{\"method\": \"mtp\", \"num_speculative_tokens\": 3}",
    "--max-model-len", "16384", "--gpu-memory-utilization", "0.9"
  ],
  "isServerless": false,
  "isPublic": false
}
```

**paddleocr-vl.json**:
```json
{
  "name": "paddleocr-vl16-vllm",
  "imageName": "vllm/vllm-openai:latest",
  "category": "NVIDIA",
  "containerDiskInGb": 50,
  "volumeInGb": 30,
  "volumeMountPath": "/root/.cache/huggingface",
  "ports": ["8080/http"],
  "dockerStartCmd": [
    "vllm", "serve", "PaddlePaddle/PaddleOCR-VL-1.6",
    "--port", "8080", "--trust-remote-code",
    "--served-model-name", "paddleocr-vl",
    "--max-model-len", "16384", "--gpu-memory-utilization", "0.85"
  ],
  "isServerless": false, "isPublic": false
}
```
When launching the pod/endpoint from this template, pin Ampere+ GPUs (`gpuTypeIds`) and, if needed, `allowedCudaVersions` — CC < 8.0 machines will fail.

**deepseek-ocr.json**:
```json
{
  "name": "deepseek-ocr-vllm",
  "imageName": "vllm/vllm-openai:latest",
  "category": "NVIDIA",
  "containerDiskInGb": 50,
  "volumeInGb": 30,
  "volumeMountPath": "/root/.cache/huggingface",
  "ports": ["8080/http"],
  "dockerStartCmd": [
    "vllm", "serve", "deepseek-ai/DeepSeek-OCR",
    "--port", "8080", "--trust-remote-code",
    "--served-model-name", "deepseek-ocr",
    "--logits-processors", "vllm.model_executor.models.deepseek_ocr:NGramPerReqLogitsProcessor",
    "--no-enable-prefix-caching", "--mm-processor-cache-gb", "0",
    "--gpu-memory-utilization", "0.9"
  ],
  "isServerless": false, "isPublic": false
}
```

**unlimited-ocr.json** — note the dedicated image; target H100 (`gpuTypeIds: ["NVIDIA H100 80GB HBM3"]`) at launch:
```json
{
  "name": "unlimited-ocr-vllm",
  "imageName": "vllm/vllm-openai:unlimited-ocr",
  "category": "NVIDIA",
  "containerDiskInGb": 60,
  "volumeInGb": 40,
  "volumeMountPath": "/root/.cache/huggingface",
  "ports": ["8000/http"],
  "dockerStartCmd": [
    "baidu/Unlimited-OCR",
    "--trust-remote-code",
    "--logits_processors", "vllm.model_executor.models.unlimited_ocr:NGramPerReqLogitsProcessor",
    "--no-enable-prefix-caching", "--mm-processor-cache-gb", "0"
  ],
  "isServerless": false, "isPublic": false
}
```

**dots-ocr.json**:
```json
{
  "name": "dots-ocr-vllm",
  "imageName": "vllm/vllm-openai:latest",
  "category": "NVIDIA",
  "containerDiskInGb": 50,
  "volumeInGb": 30,
  "volumeMountPath": "/root/.cache/huggingface",
  "ports": ["8080/http"],
  "dockerStartCmd": [
    "vllm", "serve", "rednote-hilab/dots.ocr",
    "--port", "8080", "--trust-remote-code",
    "--served-model-name", "dots-ocr",
    "--gpu-memory-utilization", "0.9", "--max-num-seqs", "8"
  ],
  "isServerless": false, "isPublic": false
}
```

**mineru.json** — custom image needed (build `FROM vllm/vllm-openai` + `pip install "mineru[vllm]"`, CMD `mineru-openai-server --port 8000`), then:
```json
{
  "name": "mineru-server",
  "imageName": "yourregistry/mineru-vllm:2.5",
  "category": "NVIDIA",
  "containerDiskInGb": 60,
  "volumeInGb": 40,
  "volumeMountPath": "/root/.cache",
  "ports": ["8000/http"],
  "dockerStartCmd": [],
  "isServerless": false, "isPublic": false,
  "containerRegistryAuthId": "YOUR_REGISTRY_AUTH_ID"
}
```

### Launching from a template
```bash
# Pod (dev/interactive)
curl -X POST https://api.runpod.io/v2/pods \
  -H "Authorization: Bearer $RUNPOD_API_KEY" -H "Content-Type: application/json" \
  -d '{"name":"glm-ocr-pod","templateId":"TEMPLATE_ID","gpuTypeIds":["NVIDIA L4"],"gpuCount":1}'

# Serverless endpoint (production, scale-to-zero) — set isServerless:true on the template first
curl -X POST https://api.runpod.io/v2/endpoints \
  -H "Authorization: Bearer $RUNPOD_API_KEY" -H "Content-Type: application/json" \
  -d '{"name":"glm-ocr-ep","templateId":"TEMPLATE_ID","gpuTypeIds":["NVIDIA L4"],
       "workersMin":0,"workersMax":3,"scalerType":"QUEUE_DELAY","scalerValue":4,
       "flashboot":true,"idleTimeout":5}'
```

### Serverless caveats worth knowing
1. **Raw `vllm serve` templates suit Pods; Serverless wants the queue protocol.** For serverless, either use RunPod's prebuilt **vLLM worker** (`runpod/worker-v1-vllm`, set `MODEL_NAME=zai-org/GLM-OCR` + `VLLM_ARGS` via env — works for GLM-OCR, dots.ocr, PaddleOCR-VL) or wrap your server in a thin `runpod.serverless` handler. DeepSeek-OCR and Unlimited-OCR need custom images anyway because of the logits-processor flags.
2. **Network volume for weights** (`networkVolumeId` on the endpoint) — avoids re-downloading 2–7 GB weights on every cold start; combine with `flashboot`.
3. **Cold starts**: vLLM CUDA-graph capture takes 1–3 min for the 3B models; for latency-sensitive KIE keep `workersMin: 1` on GLM-OCR (it's cheap) and scale-to-zero only the heavy models.
4. **API v2 is public beta** — schema is stable per the OpenAPI spec, but regenerate your client from `/v2/openapi.json` periodically; v1 (`rest.runpod.io/v1`) remains available as fallback.

---

## Suggested mapping (RunPod GPU inventory → model)

| Workload | Model | RunPod GPU pick | Why |
|---|---|---|---|
| Invoices/receipts, KIE, high-QPS | GLM-OCR | L4 or RTX 4090 (cheap), serverless min=1 | 0.9B + MTP = best $/page |
| Multilingual / messy scans | PaddleOCR-VL-1.6 | L4 / A10 | Ampere floor; pipeline CPU-heavy — pick high-vCPU flavors |
| Long contracts, books, 40+ pages | Unlimited-OCR | **H100 80GB** (SGLang FA3) | R-SWA flat KV + 32K one-shot |
| Bulk clean-PDF digitization | DeepSeek-OCR | A100 / L40S spot | ~200k pages/day/GPU; retry budget for loops |
| Forms needing JSON + bboxes | dots.ocr | A100 40GB | VRAM-hungry at batch; accuracy over speed |
| Mixed multi-format ingestion | MinerU | L4 (vlm) or CPU flavor (pipeline) | Platform, not just a model |
