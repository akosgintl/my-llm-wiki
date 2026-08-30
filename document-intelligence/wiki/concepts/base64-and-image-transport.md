---
aliases: ["Base64 and Image Transport"]
tags: [performance, serving, pipeline, optimization]
sources: [ocr-rasterize-encoding-bottlenecks.md, ocr-vdu-complete-study.md]
created: 2026-08-27
updated: 2026-08-27
---

# Base64 and Image Transport

## The cargo cult, measured

"Base64 is slow." **It is not.** Python's `base64` (binascii, C) encodes at ~1–2 GB/s. A 250 KB JPEG encodes in ~0.2 ms; across 200 pages that is ~40 ms of CPU, parallelizable. **Base64 encoding is never your bottleneck.**

## The four real costs hiding next to it

1. **+33% wire size.** 200 × 250 KB = 50 MB → 67 MB. On a 10–40 Gbps cluster network: tens of ms aggregate, irrelevant. Over WAN/VPN to a remote GPU (**[[RunPod]]**): 17 MB extra at 100 Mbps ≈ **1.4 s** — now it is real.
2. **JSON handling of megabyte strings.** `json.dumps` on a dict containing a 350 KB string copies it several times, and stdlib `json` also validates it as UTF-8. Fix: **orjson** (5–10× faster on large payloads) and build the data URI with a single pre-allocated `b"".join([prefix, b64, suffix])` — never f-strings on megabyte strings.
3. **Server-side buffering.** The [[vLLM]] API server reads the whole body, parses JSON, base64-decodes and PIL-decodes — all CPU in the frontend process. This is the reason `--api-server-count N` exists and why GPU pods need 8–16 vCPU, not the K8s default 2.
4. **Memory churn.** 300 in-flight requests × ~1 MB payload × several copies each = transient GBs and GC pressure in a Python gateway.

## The three transports, ranked for an in-cluster topology

**`file://` local path** (`vllm serve --allowed-local-media-path /shared`) — the gateway writes JPEG to a volume both containers see; the request carries a path, not pixels. Zero encode, zero body bloat, zero server-side decode. **Best as a sidecar pattern**: gateway container + vLLM container in one pod sharing an `emptyDir` on node NVMe. **Do not do cross-node file passing** — that forces RWX storage (Azure Files) whose per-file latency costs more than base64 saved.

**`image_url` (http)** — point vLLM at blob storage (SAS URL) or an in-cluster nginx. Offloads bytes from the request path and lets vLLM fetch in parallel; adds a fetch RTT and new failure modes (timeouts, `--allowed-media-domains`). **The right choice for remote GPUs**: upload pages to blob once, send tiny requests.

**Inline base64** — still the right default *in-cluster*: simplest, stateless, and with orjson plus turbo-JPEG at 1024 px the total gateway-side overhead is ~1–3 ms per page. **Optimize it, do not eliminate it.**

## Non-bottlenecks people optimize anyway

Base64 encode CPU; PNG-vs-JPEG *quality* debates below 1024 px (resolution dominates quality); HTTP/2 versus 1.1 in-cluster; gzip on JPEG bodies (already compressed — it only burns CPU).

## The ranked bottleneck audit

Where the 200-page wall-clock actually goes:

1. **GPU inference** — dominant *if everything else is done right*. Everything else exists to stay under this line.
2. **Wrong raster resolution** — the most common self-inflicted 2–5×. See [[Rasterization at Model-Native Resolution]].
3. **vLLM server-side CPU preprocessing.** Watch signal: GPU utilization sawtoothing below 90% while the gateway is idle.
4. **Serial or thread-based Python rasterization** — costs 10–20×.
5. **Source PDF download.** A 200-page scan is 100–400 MB, and PDFs are not streamable (xref at EOF) — you need the whole file before page 1 renders. Parallelize with ranged GETs and start the qpdf split the instant it lands.
6. **Straggler pages** — 3–5 loop/dense-table pages define your p100.
7. **Event-loop blocking** — doing b64/JSON/PIL inline in asyncio handlers freezes *all* in-flight requests; at 250 in-flight this compounds into seconds of p99.
8. **HTTP plumbing** — httpx's silent default `max_connections=100` caps a 250-request wave; missing keep-alive means a TLS handshake per request; a mesh mTLS sidecar encrypting 67 MB of base64 costs real CPU.
9. **Queue hop latency** — Azure Service Bus adds ~20–100 ms per message. Fine for job intake, wrong for 200 per-page messages on the hot path. Enqueue the *document*, fan out pages in-process.
10. **Needless temp files** — keep pages as in-memory `BytesIO` unless deliberately using the `file://` sidecar.

## Which language should run this

The gateway's job — high fan-out async HTTP, CPU-parallel raster/encode, zero event-loop stalls, ordered fan-in, tight memory — is exactly what Python is worst at.

| Language | Fit |
|---|---|
| **Rust** (tokio + rayon, `pdfium-render`, mozjpeg, axum) | Best raw fit; ~10–50 MB RSS versus Python's GBs at high fan-out. What the [[neural-maze production-ocr-course]] uses. Cost: team ramp-up |
| **Go** (goroutines, `go-pdfium`) | 80% of the benefit at 30% of the effort |
| **.NET 8+** (async/await, Channels, TPL Dataflow, PDFium bindings) | **The Azure-shop answer** — first-class Service Bus/Blob/Identity SDKs may beat Go's marginal perf edge on AKS |
| Python (kept honest) | asyncio + `ProcessPoolExecutor` + orjson + uvloop. Fine to prototype and even ship v1 |

**Hybrid, not rewrite:** keep Python where the ML ecosystem lives (pipeline drivers, eval harnesses); put the document gateway in Rust/Go/.NET. The `pdftoppm`-subprocess trick makes rasterization language-neutral, which shrinks the decision to "who runs the async HTTP hot path best."

## Related

[[Rasterization at Model-Native Resolution]] · [[Document Fan-Out and Fan-In]] · [[vLLM]] · [[RunPod]]
