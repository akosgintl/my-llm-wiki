---
aliases: ["IBM Research"]
tags: [organization]
sources: [Local PDF-Markdown and Document Parsing - State of the Art (August 2026).md]
created: 2026-08-27
updated: 2026-08-27
---

# IBM Research

Publisher of [[Docling]] (MIT) and [[Granite-Docling]] (Apache-2.0) — the framework-and-model pairing that anchors the *permissively licensed, enterprise-integrated* corner of the landscape.

Their distinguishing strategic choice is **avoiding OCR where the PDF already has a text layer**: the classic Docling pipeline matches layout predictions back to the PDF's own cells rather than re-transcribing. Peter Staar (Principal Research Staff Member, IBM Research Zurich; Chair of the Docling Technical Steering Committee at the Linux Foundation) states this "reduces errors, and it also speeds up the time-to-solution by 30 times."

Docling was **donated to the LF AI & Data Foundation in April 2025**, which is what makes it a safe long-term dependency in a way a single-vendor framework is not. Enterprise integration runs through Red Hat OpenShift Operator, RHEL AI and OpenShift AI.

The trade-off: their models are English-first and small (258M), so absolute accuracy on hard tables and formulas trails the 1–2B specialists, and the classic pipeline scores poorly on [[OmniDocBench]] partly for metric reasons and partly for real ones.

**Source-quality note:** one vendor page (idp-software.com) misdated Granite-Docling to "January 2026" and mischaracterized the LF donation. Primary IBM/HF/LF sources confirm Granite-Docling shipped **2025-09-17** and the donation was **April 2025**.

## Related

[[Docling]] · [[Granite-Docling]]
