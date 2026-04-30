# FinOps & Capacity Planning: Receipt Parser (VERA)

This document projects the infrastructure and LLM costs for the `receipt_parser` module.

> 💲 **Pricing source-of-truth:** [`src/app/core/pricing_registry.py`](https://github.com/JMMAILabs/frugal-fortress-architecture) — last verified 2026-04. The values below are mirrors of that registry; if they diverge, the registry wins.
 
## 1. Baseline Assumptions
*   **1 Receipt Image** ≈ 1,120 tokens (Standard VLM image encoding).
*   **JSON Output** ≈ 500 tokens.
*   **Target Volume:** 10,000 receipts processed per month.
*   **Self-Correction Loop:** Assume 10% of receipts require 1 retry due to math validation failures.

## 2. LLM Provider Cost Comparison (Per Receipt)

| Provider / Model | Tier | Input Cost (Image) | Output Cost (JSON) | Total Cost (per receipt) | Total Cost (10k receipts) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **LlamaParse + Groq** (`llama-3.3`) | Free | $0.0000 (Free Tier) | $0.0004 | **$0.0004** | **$4.00** |
| **Vertex AI** (`gemini-2.5-flash-lite`) | Premium | $0.00011 | $0.00020 | **$0.00031** | **$3.10** |
| **Vertex AI** (`gemini-2.5-flash`) | Pro | $0.00033 | $0.00125 | **$0.00158** | **$15.80** |

## 3. Architectural Savings (The "Frugal" Impact)
1. **Edge Image Compression:** By using the HTML5 Canvas API on the Next.js PWA, we resize images to a max of 1024px before uploading. This reduces payload size from ~5MB to ~150KB, saving massive bandwidth costs and preventing VLM token explosion.
2. **Deterministic Math Validation:** Instead of asking the LLM to "check its math" (which consumes tokens and is unreliable), we use a pure Python validator (`validate_receipt_math`). We only spend retry tokens when a mathematical error is strictly proven.