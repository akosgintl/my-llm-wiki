# Rasterization, Encoding & the Real Bottlenecks — a Critical Pass
*Companion to the API-call strategy doc (Aug 2026). Numbers marked ~ are ballpark engineering figures — benchmark on your own document mix before hardcoding.*

---

## 0. The critique that matters most, up front

**Most OCR pipelines rasterize at the wrong resolution and then argue about base64.** The canonical mistake: render at 300 DPI "for quality" (A4 → 2480×3508 px), ship ~2–8 MB per page, and let the model's preprocessor immediately downscale to its native 1024–1600 px input. That's 4–9× wasted pixels through the *entire* chain — render time, encode time, transfer bytes, server-side decode, resize — for zero accuracy gain, because the model never sees the extra pixels.

**Rule 1: render to the model's native input size, not to a DPI.**
- DeepSeek-OCR / Unlimited-OCR base mode: 1024 px long edge (Gundam tiles from 640).
- PaddleOCR-VL / GLM-OCR: dynamic-resolution encoders, but their pipelines crop regions from a page image ~1200–1600 px long edge is plenty.
- Practically: target **long edge 1024–1600 px** (≈110–150 DPI for A4). Only exceed this for known tiny-print documents, and then prefer the model's tiling mode over brute DPI.

Everything below assumes Rule 1. It alone typically cuts gateway CPU and transfer volume by ~75–85%.

---

## 1. Rasterization tools — matrix

| Tool | Type | License | Speed (~pages/s/core @ ~150 DPI) | Notes |
|---|---|---|---|---|
| **PyMuPDF (fitz)** | Python lib (MuPDF) | **AGPL-3.0** (or paid commercial) | ~20–50 | Fastest Python-native; `Matrix(zoom)` renders directly at target px. **Not thread-safe → process pool.** License is the real problem (see below) |
| **pypdfium2 / PDFium** | Python lib (Chrome's PDFium) | **Apache-2.0/BSD** | ~15–40 | The permissive-license workhorse. Marginally slower than MuPDF on some docs, irrelevant in parallel |
| **pdftoppm (Poppler)** | CLI | GPL-2 (process boundary = safe) | ~10–25 | The classic native Linux answer. Rock-solid, trivially parallel via subprocess |
| **pdftocairo (Poppler)** | CLI | GPL-2 | ~10–25 | Same engine, better anti-aliasing, direct JPEG/PNG out |
| **mutool draw (MuPDF)** | CLI | AGPL-3.0 | ~20–50 | Fastest CLI; AGPL via subprocess of an *unmodified* binary is generally considered safe, but get legal sign-off |
| **libvips (pdfload)** | Lib/CLI | LGPL | ~15–30 | Streams, tiny memory footprint; uses poppler/pdfium backend. Shines when you also resize/convert in one pass |
| Ghostscript | CLI | AGPL-3.0 | ~3–10 | Slower; only if you need its color management/PDF repair |
| ImageMagick `convert` | CLI | — | ~1–5 | ❌ Delegates to Ghostscript with overhead. Never for production |
| **qpdf** | CLI (not a rasterizer) | Apache-2.0 | ~100+ splits/s | Splits a 200-page PDF into per-page PDFs in <1 s — the enabler for CLI-parallel rendering |

**License critique you should not skip:** your current DeepSeek-OCR toolchain already drags in PyMuPDF (AGPL) — flagged in their own GitHub issue #223 — and PyMuPDF is everyone's default. For a commercial IDP product, standardize on **pypdfium2 (Python)** or **pdftoppm (CLI)** and treat MuPDF-family as opt-in-with-legal-review. This is a cheap decision now and an expensive one later.

### Commands / snippets

**Poppler, one page per process (the simple robust baseline):**
```bash
# Render page N only, ~135 DPI, JPEG out — one invocation per page, GNU parallel across cores
seq 1 200 | parallel -j "$(nproc)" \
  'pdftoppm -jpeg -jpegopt quality=90 -r 135 -f {} -l {} doc.pdf out/page_{}'
```

**qpdf split → parallel render (best when one worker = one page-file, e.g. K8s Jobs):**
```bash
qpdf --split-pages doc.pdf 'pages/pg-%d.pdf'        # <1 s for 200 pages
ls pages/*.pdf | parallel -j "$(nproc)" 'pdftoppm -jpeg -r 135 {} {.}'
```

**PyMuPDF, render-at-target-size, process pool (if AGPL is cleared):**
```python
import fitz, concurrent.futures as cf

TARGET_LONG_EDGE = 1024

def render_page(args):
    path, pno = args
    with fitz.open(path) as doc:            # open per process — fitz is not thread-safe
        page = doc[pno]
        zoom = TARGET_LONG_EDGE / max(page.rect.width, page.rect.height)
        pix = page.get_pixmap(matrix=fitz.Matrix(zoom, zoom), colorspace=fitz.csRGB)
        return pno, pix.tobytes("jpeg", jpg_quality=90)   # encode inside the worker

with cf.ProcessPoolExecutor() as ex:        # processes, NOT threads (GIL + thread-safety)
    pages = dict(ex.map(render_page, [("doc.pdf", i) for i in range(200)], chunksize=8))
```

**pypdfium2 equivalent (permissive license):**
```python
import pypdfium2 as pdfium
pdf = pdfium.PdfDocument("doc.pdf")                     # per process
page = pdf[pno]
scale = TARGET_LONG_EDGE / max(page.get_size())
bitmap = page.render(scale=scale)
pil = bitmap.to_pil()                                   # then JPEG-encode q90
```

**Timing reality check:** 200 pages ÷ (16 cores × ~20 pages/s/core) ≈ **0.6–1.5 s** wall-clock at target resolution. Done serially at 300 DPI it's 60–120 s. Rasterization is only a bottleneck when you do it wrong; done right it disappears under the GPU time.

**One caveat on process-per-page with big scanned PDFs:** each worker re-opens the file and parses the xref. For a 300 MB scanned PDF × 16 workers that's real I/O. Mitigate with page-range chunks per worker (worker takes pages 1–13, one open) or the qpdf split (each worker touches only its ~1.5 MB slice).

---

## 2. Output format & encoding — what actually moves the needle

| Choice | Size/page (~A4 @1024px) | Encode cost | OCR impact | Verdict |
|---|---|---|---|---|
| PNG (default zlib-6) | ~300–800 KB | Slow (zlib) | None (lossless) | ❌ Slowest encode, biggest bytes |
| PNG compress=1 | ~400–900 KB | Fast | None | Meh — still big |
| **JPEG q90 (libjpeg-turbo)** | ~120–300 KB | Very fast (SIMD) | Negligible at ≥1024 px | ✅ **Default** |
| JPEG q75 | ~60–150 KB | Very fast | Ringing artifacts around small glyphs — measurable CER hit on fine print | Only for previews |
| Grayscale JPEG q90 | ~40% smaller | Fast | Fine for B/W scans; models convert to RGB internally anyway | ✅ For scan-only corpora |
| WebP lossless | ~PNG×0.7 | Slow | None | Not worth the encode CPU here |

Two server-side reasons JPEG wins beyond size: vLLM's preprocessor must **decode** every image on CPU before the GPU sees it, and JPEG decode (libjpeg-turbo, SIMD) is several times faster than PNG inflate — at 200 concurrent requests that decode CPU is *your* p99. Ensure `libjpeg-turbo` is actually the backend in both gateway and vLLM images (it is in standard Pillow wheels; verify with `PIL.features`). If you do heavy resize work gateway-side, use **pyvips or OpenCV**, not vanilla PIL — SIMD resize is 3–10× faster.

---

## 3. Base64 — the honest accounting

**The cargo-cult claim:** "base64 is slow." **The measurement:** Python's `base64` (binascii, C) encodes at ~1–2 GB/s. A 250 KB JPEG encodes in ~0.2 ms. Across 200 pages: ~40 ms of CPU, parallelizable. Base64 *encoding* is never your bottleneck.

**The real costs hiding next to it:**
1. **+33% wire size.** 200 × 250 KB = 50 MB → 67 MB. On a 10–40 Gbps cluster network: tens of ms aggregate — irrelevant. Over WAN/VPN to a remote GPU (RunPod!): 17 MB extra at 100 Mbps = ~1.4 s — now it's real.
2. **JSON handling of megabyte strings.** `json.dumps` on a dict containing a 350 KB string copies it multiple times; stdlib `json` also decodes/validates it as UTF-8. Fix: **orjson** (5–10× faster on large payloads) and build the data-URI with a single pre-allocated `b"".join([prefix, b64, suffix])` — no f-strings on megabyte strings.
3. **Server-side buffering.** The vLLM API server reads the whole body, parses JSON, base64-decodes, PIL-decodes — all CPU in the frontend process. This is the #1 reason `--api-server-count N` exists and why GPU pods need generous vCPU (8–16 for a busy replica, not the K8s-default 2).
4. **Memory churn.** 300 in-flight requests × ~1 MB payload copies × several copies each = transient GBs and GC pressure in a Python gateway.

**Alternatives to inline base64, ranked for your topology:**

- **`file://` local path** (`vllm serve ... --allowed-local-media-path /shared`): gateway writes JPEG to a volume both containers see; request carries a path, not pixels. Zero encode, zero body bloat, zero server-side b64-decode. **Best as a sidecar pattern** (gateway container + vLLM container in one pod, shared `emptyDir` on node NVMe). Across nodes it forces RWX storage (Azure Files) whose per-file latency will *cost* you more than base64 saved — don't do cross-node file passing.
- **`image_url` (http)**: point vLLM at your blob store (SAS URL) or an in-cluster nginx serving the rasterized pages. Offloads bytes from the request path and lets vLLM fetch in parallel; adds a fetch RTT per image and a new failure mode (fetch timeouts, `--allowed-media-domains`). Good for RunPod remote GPUs — upload pages to blob once, send tiny requests.
- **Inline base64**: still the right default *in-cluster* — simplest, stateless, and with orjson + turbo-JPEG at 1024 px the total overhead per page is ~1–3 ms gateway-side. Optimize it, don't eliminate it.

---

## 4. Full bottleneck audit — ranked, with the uncomfortable ones

Where the 200-page wall-clock actually goes, in order:

1. **GPU inference** (~60–120 s single H100 wave) — dominant *if everything else is done right*. Everything else exists to stay under this line.
2. **Wrong raster resolution** (see §0) — the most common self-inflicted 2–5× on the entire upstream.
3. **vLLM server-side CPU preprocessing** — JSON+b64+image decode+resize for 200 concurrent multimodal requests in one Python frontend. Mitigations: `--api-server-count`, big vCPU allocation, JPEG not PNG, pre-resized images. **Watch signal:** GPU utilization sawtoothing below 90% while gateway is idle → the frontend is starving the engine.
4. **Serial or thread-based Python rasterization** — GIL + fitz thread-unsafety make threading useless; forgetting `ProcessPoolExecutor` costs 10–20×.
5. **Source PDF download.** A 200-page *scanned* PDF is 100–400 MB. From Azure Blob in-region: 1–3 s, fine. From a user upload over WAN: possibly longer than inference. Also: PDFs aren't streamable (xref at EOF) — you need the whole file before page 1 renders. Mitigation: rasterize *as the tail arrives* is not possible; instead parallelize download (ranged GETs) and start qpdf split the instant it lands.
6. **Straggler pages** — 3–5 repetition-loop / dense-table pages define your p100. Hedged re-issue (from the previous doc) is worth 20–30%.
7. **Event-loop blocking in the gateway** — doing b64/JSON/PIL inline in asyncio handlers freezes *all* in-flight requests for ms at a time; at 250 in-flight this compounds into seconds of added p99. Push CPU work to a process pool or out of Python entirely (§5).
8. **HTTP plumbing** — httpx default `max_connections=100` silently caps your 250-request wave (raise `Limits`); missing keep-alive = TLS handshake per request; a service-mesh mTLS sidecar encrypting 67 MB of b64 costs real CPU. HTTP/2 multiplexing helps head-of-line only marginally here — connection *pool size* matters more.
9. **Queue hop latency** — Azure Service Bus adds ~20–100 ms per message; fine for job intake, wrong for 200 per-page messages on the hot path. Enqueue the *document*, fan out pages in-process (or Redis if you must distribute).
10. **Writing temp files** you didn't need — keep pages as in-memory `BytesIO` unless deliberately using the `file://` sidecar pattern; container overlayfs writes are slow and pod-ephemeral-storage-limited.

Non-bottlenecks people optimize anyway: base64 *encode* CPU (§3), PNG-vs-JPEG *quality* debates below 1024 px (resolution dominates quality), HTTP/2 vs 1.1 in-cluster, and gzip on JPEG bodies (already compressed — disable it, it only burns CPU).

---

## 5. Beyond Python — who should run this gateway?

The gateway's job description: high fan-out async HTTP, CPU-parallel raster/encode, zero event-loop stalls, ordered fan-in, tight memory. Python *can* do it (subprocess CLI rasterizers make the GIL irrelevant for rendering), but the hot path — encode → JSON build → HTTP → parse → reassemble at 250 in-flight — is where Python taxes you.

| Language | Async / concurrency | PDF+image story | Fit |
|---|---|---|---|
| **Rust** | tokio (async I/O) + rayon (CPU parallel) in one process, no GIL, no GC | `pdfium-render` (Apache PDFium), `image`/`mozjpeg`/`turbojpeg` bindings, SIMD `base64` crate, axum/hyper/reqwest | **Best raw fit.** One static binary doing raster+encode+fan-out at wire speed; ~10–50 MB RSS vs Python's GBs at high fan-out. This is why the public AKS production-OCR reference (neural-maze) uses a Rust/Axum gateway. Cost: team ramp-up |
| **Go** | goroutines make 250-way fan-out trivial; excellent stdlib HTTP | `go-pdfium` (cgo→PDFium, permissive), `go-fitz` (cgo→MuPDF, AGPL again — avoid), fast native JPEG | **Best pragmatism/perf ratio.** 80% of Rust's benefit at 30% of the effort; cgo overhead is noise at page granularity |
| **C# / .NET 8+** | mature async/await, Channels, TPL Dataflow for the fan-out/fan-in graph | `Docnet`/`PDFtoImage` (PDFium), SkiaSharp/ImageSharp; **first-class Azure SDKs** (Service Bus, Blob, Identity) | **The Azure-shop answer.** On AKS with Service Bus + Blob already in play, .NET's ecosystem fit may beat Go's marginal perf edge. Don't dismiss it for being unfashionable |
| Node/Bun | fine async, but CPU work needs worker_threads juggling | pdfium bindings exist, patchy | Only if the team is JS-native |
| Elixir/BEAM | superb concurrency model | no serious native PDF/imaging — shells out to CLI | Elegant, wrong toolbox |
| Python (kept honest) | asyncio + `ProcessPoolExecutor` + orjson + uvloop | everything, instantly | Fine to **prototype and even ship v1** — with the discipline of §4.7. Revisit when gateway CPU or p99 jitter shows up in traces |

**Recommended shape — hybrid, not rewrite:**
- Keep **Python where the ML ecosystem lives**: the glmocr/PaddleOCR pipeline drivers, MinerU, evaluation harnesses.
- Put the **document gateway in Rust (max) or Go/.NET (pragmatic)**: download → qpdf/PDFium split → parallel render @1024 → turbo-JPEG q90 → semaphored fan-out → hedged retries → ordered fan-in. This component is pure systems programming with zero ML dependencies — exactly what Python is worst at and Rust/Go/.NET are built for.
- The CLI-subprocess trick (`pdftoppm` via GNU parallel) is the *language-neutral* escape hatch: it makes rasterization equally fast from any orchestrator, shrinking the language decision to "who runs the async HTTP hot path best" — which is precisely where Rust/Go/.NET win.

---

## TL;DR
1. Render at model-native resolution (long edge ~1024–1600 px), never blanket 300 DPI — the single biggest optimization in the whole pipeline.
2. Tools: pypdfium2 (lib) or pdftoppm (CLI) for permissive licensing; PyMuPDF/mutool are fastest but AGPL — legal sign-off first. qpdf split enables clean CLI parallelism. Process pools, never threads.
3. JPEG q90 via libjpeg-turbo; grayscale for scans; PNG only if lossless is a compliance requirement.
4. Base64 encode is a non-issue; the real costs are +33% wire (matters WAN/RunPod, not in-cluster), JSON handling of megabyte strings (orjson), and vLLM frontend CPU (`--api-server-count`, big vCPU). `file://` sidecar or blob-URL fetch are the escape hatches when those bite.
5. Biggest hidden bottlenecks: server-side preprocessing CPU, event-loop blocking, httpx connection caps, per-page queue hops, straggler pages.
6. Gateway language: Rust if you're investing, Go or .NET 8 for pragmatism (with .NET's Azure SDKs a genuine argument on AKS), Python acceptable for v1 with process pools + orjson + uvloop. Keep Python for the ML pipeline layer regardless.
