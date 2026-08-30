---
aliases: ["LiteLLM"]
tags: [serving, load-balancing, infrastructure, tool]
sources: [ocr-api-call-strategy.md, ocr-vdu-complete-study.md, ocr-language-models-deployment-critical-pass-2.md]
created: 2026-08-27
updated: 2026-08-27
---

# LiteLLM

OpenAI-compatible proxy used as the pragmatic load balancer and the multi-cloud seam.

## As a load balancer

Tier 2 in the [[Load Balancing Inference Pools]] ranking, behind the Kubernetes Gateway API Inference Extension but ahead of a service mesh. What it gives you:

- per-model routing (`model: glm-ocr` → the GLM pool)
- least-busy / lowest-latency balancing strategies
- health checks, automatic retries and fallbacks (e.g. [[dots.ocr]] overflow → [[GLM-OCR]])
- budgets and rate limits per caller

Runs as a simple Deployment; the config is a YAML list of replica URLs. It is the right choice while the fleet is under ~20 replicas, and the *only* practical choice on [[RunPod]] where you do not control the ingress layer.

## As the multi-cloud seam

Because every model in the architecture is plain [[vLLM]] behind an OpenAI API, moving a workload between AKS, an EU-sovereign provider and RunPod is **a LiteLLM route entry, not a re-architecture**. That portability is the payoff of [[Pipeline as Platform, Model as Config]] and of the three-tier posture in [[EU Sovereign GPU Hosting]].

## Related

[[Load Balancing Inference Pools]] · [[vLLM]] · [[RunPod]] · [[EU Sovereign GPU Hosting]]
