---
aliases: ["EU Sovereign GPU Hosting"]
tags: [infrastructure, compliance, gpu-cloud, cost]
sources: [ocr-language-models-deployment-critical-pass-2.md, ocr-vdu-complete-study.md]
created: 2026-08-27
updated: 2026-08-27
---

# EU Sovereign GPU Hosting

For a pipeline processing production documents, **the axis that matters is jurisdiction, not price**.

## The fine print people get wrong

**"EU region of a US cloud" ≠ sovereign.** CLOUD Act exposure follows **incorporation**, not server location. Azure's EU regions remain US-compellable.

Even non-US incorporation needs auditing:

- **Scaleway** (French, Iliad) — strongest H100 availability in the sovereign class, ~€0.50–2.50/hr, managed Kubernetes (Kapsule) with GPU pools. The closest like-for-like AKS substitute; SecNumCloud in progress.
- **OVHcloud** — multi-region EU, enterprise SLAs — **but SecNumCloud does not cover the GPU SKUs**, and a **2026 Ontario ruling compelled its Canadian entity to hand over data stored in France**. Audit corporate structures, not marketing.
- **Hetzner** — cheapest, German, but **no A100/H100 class**; workstation-RTX only. Fine for a 0.9–4B replica, not the H100 tier.
- **Nebius** — EU datacenters, competitive H100 pricing, but structurally complex (ex-Yandex, multi-jurisdiction). Legal review required; do not assume it matches OVHcloud or Hetzner.
- Also: Gcore, Verda (ex-DataCrunch), STACKIT, IONOS, Exoscale.

**2026 reality check:** OVHcloud, Scaleway *and* Hetzner all raised prices in one quarter (Hetzner +30–37%, citing DRAM +171% YoY). **The EU-discount era is ending — re-quote before budgeting.**

## The four market shapes

| Shape | Who | Fit |
|---|---|---|
| Hyperscale neoclouds | CoreWeave (Platinum ClusterMAX, priced near hyperscalers), Nebius, Lambda, Crusoe, Nscale, Fluidstack | Sustained capacity ~10–15% under CoreWeave; real managed K8s so manifests port |
| Serverless GPU | [[RunPod]], Modal (Python-native, per-second), Koyeb (serverless *containers*, cleanest map onto existing vLLM images), Baseten, fal.ai | Bursty waves — the 200-page scenario |
| Marketplaces | Vast.ai, Spheron | Cheapest anywhere, host reliability varies. **Benchmarking only — never customer documents** |
| **EU-sovereign** | Scaleway, OVHcloud, Hetzner, Gcore, Verda, STACKIT, IONOS, Exoscale | **The production-document tier** |

## The three-tier posture

1. **AKS = control plane + steady state** — Service Bus, Blob, identity, the layout tier, and baseline GPU capacity stay put.
2. **Scaleway/OVHcloud = sovereign GPU burst** for production documents.
3. **RunPod/Modal/Vast = non-sensitive tier** — bake-offs, synthetic data, load tests.

## Why this is cheap to do

The connective tissue is that **everything is plain [[vLLM]] in a container behind an OpenAI API**. Multi-cloud is therefore a [[LiteLLM]] route entry, not a re-architecture — which was the point of building it that way. See [[Pipeline as Platform, Model as Config]].

## Related

[[RunPod]] · [[LiteLLM]] · [[Pipeline as Platform, Model as Config]] · [[Permissive Licensing Constraints]]
