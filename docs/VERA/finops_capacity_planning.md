# FinOps & Capacity Planning: Receipt Parser (VERA)

This document projects the infrastructure and LLM costs for the `receipt_parser` module. Gemini calculations use ADR-0017 text-token reference rates and are not confirmed image-modality or Vertex invoice costs.

> 💲 **Pricing source-of-truth:** [`src/app/core/pricing_registry.py`](https://github.com/JMMAILabs/frugal-fortress-architecture) — last verified 2026-08. The values below are mirrors of that registry; if they diverge, the registry wins.
 
## 1. Baseline Assumptions
*   **1 Receipt Image** ≈ 1,120 tokens (Standard VLM image encoding).
*   **JSON Output** ≈ 500 tokens.
*   **Target Volume:** 10,000 receipts processed per month.
*   **Self-Correction Loop:** Assume 10% of receipts require 1 retry due to math validation failures.

## 2. LLM Provider Cost Comparison (Per Receipt)

| Provider / Model | Tier | Input-token reference (1,120 tokens) | Output-token reference (500 tokens) | Text-rate projection (per receipt) | 10k receipts, one attempt each | 10k receipts, incl. 10% one-retry allowance |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **LlamaParse + Groq** (`gpt-oss-120b`) | Free | $0.0000 (Free Tier) | $0.0003 | **$0.0003** | **$3.00** | **$3.30** |
| **Vertex AI** (`gemini-3.1-flash-lite`) | Premium | $0.00028 | $0.00075 | **$0.00103** | **$10.30** | **$11.33** |
| **Vertex AI** (`gemini-3.5-flash-lite`) | Pro | $0.000336 | $0.00125 | **$0.001586** | **$15.86** | **$17.45** |
| **Vertex AI** (`gemini-3.8-flash`) | PAYG | $0.00084 | $0.001875 | **$0.002715** | **$27.15** | **$29.87** |

The retry column applies the documented assumption that 10% of receipts require one additional attempt. These are text-token reference projections, not established image-token prices. Validate modality pricing, routing, and behavior in Vertex staging before production. See [ADR-0017](../adr/0017-gemini-models-25-migration.md).

## 3. Architectural Savings (The "Frugal" Impact)
1. **Edge Image Compression:** By using the HTML5 Canvas API on the Next.js PWA, we resize images to a max of 1024px before uploading. This reduces payload size from ~5MB to ~150KB, saving massive bandwidth costs and preventing VLM token explosion.
2. **Deterministic Math Validation:** Instead of asking the LLM to "check its math" (which consumes tokens and is unreliable), we use a pure Python validator (`validate_receipt_math`). We only spend retry tokens when a mathematical error is strictly proven.