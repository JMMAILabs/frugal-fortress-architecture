# Tiering Strategy & FinOps Capacity Planning

This document defines the operational limits, model routing, and break-even pricing analysis for the Receipt Parser (VERA) module.

> 💲 **Pricing source-of-truth:** [`src/app/core/pricing_registry.py`](https://github.com/JMMAILabs/frugal-fortress-architecture) — last verified 2026-04. The values below are mirrors of that registry; if they diverge, the registry wins.

## 1. Stripe Metadata Contract
Payment links must inject the following custom fields into the Stripe Checkout Session:
*   `receipt_parser_user_id`
*   `language_code`
*   `tier`

## 2. Free Tier (Third-Party Stack)
*   **Pipeline Flow:** Image/PDF -> LlamaParse -> Markdown -> PII Tokenization (Presidio) -> Groq `llama-3.3-70b-versatile` -> JSON -> ALE Encryption -> Supabase.
*   **Limits:**
    *   30 receipts / month.
    *   5 receipts / day.
    *   Maximum Storage Capacity: 100 receipts.
    *   Max 3 versions per `image_hash`.
*   **Data Retention:** 3 months.
*   **Disclaimer:** The Free tier utilizes third-party OCR/LLM APIs (LlamaParse + Groq). It is not recommended for highly sensitive financial documents (e.g., PHI/PII heavy invoices).

## 3. Paid Tiers (Vertex AI Exclusive)
To guarantee Zero Data Retention, paid tiers exclusively utilize Google Vertex AI under enterprise terms. The platform does not route any traffic through other LLM vendors.

*   **Premium:** `gemini-2.5-flash-lite`.
*   **Pro:** `gemini-2.5-flash`.
*   **PAYG:** Selectable models (`gemini-2.5-flash-lite`, `gemini-2.5-flash`, `gemini-2.5-pro`).

### Operational Limits

| Metric | Premium | Pro | PAYG |
| :--- | :--- | :--- | :--- |
| **Monthly Limit** | 500 receipts | 2,000 receipts | N/A |
| **Daily Limit** | 50 receipts | 200 receipts | 1,000 receipts |
| **Max Storage Capacity** | 2,000 receipts | 5,000 receipts | N/A |
| **Max Versions per Hash** | 10 | 50 | 40 |
| **Data Retention** | 1 year | 5 years | Inherits base tier |

*Note on Versioning:* Each persistence event counts as a row (the initial parse, and every human correction saved via `POST /receipts/history`). The internal LLM self-correction loop does *not* create new database rows.

## 4. FinOps & Break-Even Pricing Analysis

*Source of Truth: [Google Cloud Vertex AI Pricing](https://cloud.google.com/vertex-ai/generative-ai/pricing)*

**Conservative Baseline Assumptions:**
*   1 Input Image ≈ `1,120` tokens (Standard Gemini Flash Image proxy).
*   JSON Output ≈ `500` tokens.
*   **Self-Correction Loop:** Up to `3` paid attempts per receipt (`MAX_RETRIES = 3`).

### Premium Tier Analysis
*   **Cost per Receipt (Single Attempt):**
    *   `(1,120 * $0.10 / 1M) + (500 * $0.40 / 1M) = $0.000312`
*   **Worst-Case Scenario (3 Retries):** `$0.000936`
*   **Worst-Case Monthly Cost:** `500 * $0.000936 = $0.468`
*   **Recommended Break-Even Price:** `$0.99 / month`
*   **Final Retail Price:** **$2.99 / month**

### Pro Tier Analysis

*   **Cost (Gemini 2.5 Flash):**
    *   Single Attempt: `(1,120 * $0.30 / 1M) + (500 * $2.50 / 1M) = $0.001586`
    *   Worst-Case (3 Retries): **$0.004758**
*   **Worst-Case Monthly Cost:** `2,000 * $0.004758 = $9.516`
*   **Recommended Break-Even Price:** `$9.99 / month`
*   **Final Retail Price:** **$9.99 / month**
*   **Rationale:** Gemini 2.5 Flash provides multimodal accuracy at a price that lets us hold `$9.99 / month` retail with comfortable margins, even when the worst-case 2,000-receipt monthly ceiling is hit and every receipt exhausts the 3-retry self-correction loop.

## 5. Data Lifecycle
Upon subscription cancellation, the standard 90-day grace period applies. The user is downgraded to Free tier compute limits but retains full access to export their historical data. On Day 91, a FIFO pruning job truncates their storage to the Free tier limit (100 receipts).