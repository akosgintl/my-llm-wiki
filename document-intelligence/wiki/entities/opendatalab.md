---
aliases: ["OpenDataLab"]
tags: [organization]
sources: [Open-Source OCR per Visual Document Understanding Models for Self-Hosted AKS Pipelines - A Comparative Investigation (August 2026).md, ocr-arxiv-github-technical-review.md, Local PDF-Markdown and Document Parsing - State of the Art (August 2026).md]
created: 2026-08-27
updated: 2026-08-27
---

# OpenDataLab

Shanghai AI Lab affiliate. Publishes both [[MinerU]] — the leading open ingestion *platform* — and **[[OmniDocBench]]**, the benchmark the whole field is scored on.

That combination is worth noting explicitly when reading leaderboards: the organization that publishes the dominant benchmark also ships models that compete on it. It is not evidence of anything improper, but it belongs in the same mental column as vendors self-evaluating while pulling competitor numbers from a repo. See [[Benchmark Saturation]].

Their most useful published result is arguably not a model at all: **MinerU2.5-Pro reached 95.69 on OmniDocBench v1.6 with an architecture identical to MinerU2.5**, purely through data engineering. That is the strongest available argument that a data plan beats architecture chasing.

They also moved MinerU off AGPLv3 to a custom Apache-2.0-based license in April 2026, removing a real enterprise blocker.

## Related

[[MinerU]] · [[OmniDocBench]] · [[Benchmark Saturation]]
