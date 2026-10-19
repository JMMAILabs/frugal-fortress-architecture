# Tiering Strategy & FinOps Capacity Planning

This document defines the operational limits, model routing, and break-even pricing analysis for the Receipt Parser (VERA) module.

> 💲 **Pricing source-of-truth:** [`src/app/core/pricing_registry.py`](https://github.com/JMMAILabs/frugal-fortress-architecture) — last verified 2026-08. The values below are mirrors of that registry; if they diverge, the registry wins.

## 1. Stripe Metadata Contract
Payment links must inject the following custom fields into the Stripe Checkout Session:
*   `receipt_parser_user_id`
*   `language_code`
*   `tier`

## 2. Free Tier (Third-Party Stack)
*   **Pipeline Flow:** Image/PDF -> LlamaParse -> Markdown -> PII Tokenization (Presidio) -> Groq `openai/gpt-oss-120b` -> JSON -> ALE Encryption -> Supabase.
*   **Limits:**
    *   30 receipts / month.
    *   5 receipts / day.
    *   Maximum Storage Capacity: 100 receipts.
    *   Max 3 versions per `image_hash`.
*   **Data Retention:** 3 months.
*   **Disclaimer:** The Free tier utilizes third-party OCR/LLM APIs (LlamaParse + Groq). It is not recommended for highly sensitive financial documents (e.g., PHI/PII heavy invoices).

## 3. Paid Tiers (Vertex AI Exclusive)
To guarantee Zero Data Retention, paid tiers exclusively utilize Google Vertex AI under enterprise terms. The platform does not route any traffic through other LLM vendors.

*   **Premium:** `gemini-3.1-flash-lite`.
*   **Pro:** `gemini-3.5-flash-lite`.
*   **PAYG:** Selectable models (`gemini-3.1-flash-lite`, `gemini-3.5-flash-lite`, `gemini-3.8-flash`).

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

The model mapping and per-million text-token reference rates are documented in [ADR-0017](../adr/0017-gemini-models-25-migration.md). These rates do not confirm image-modality billing or project-specific Vertex invoices. Validate current prices by modality, routes, and behavior in Vertex staging before production.

**Conservative Baseline Assumptions:**
*   1 Input Image ≈ `1,120` tokens (Standard Gemini Flash Image proxy).
*   JSON Output ≈ `500` tokens.
*   **Self-Correction Loop:** Up to `3` paid attempts per receipt (`MAX_RETRIES = 3`).

### Premium Tier Analysis
*   **Cost per Receipt (Gemini 3.1 Flash-Lite text-rate reference projection, single attempt):**
    *   `(1,120 * $0.25 / 1M) + (500 * $1.50 / 1M) = $0.00103`
*   **Worst-Case Scenario (3 attempts):** `$0.00309`
*   **Worst-Case Monthly Cost:** `500 * $0.00309 = $1.545`
*   **Break-Even Price:** Not reassessed until image-modality pricing is validated; the current retail price remains unchanged.
*   **Final Retail Price:** **$2.99 / month**

### Pro Tier Analysis

*   **Cost (Gemini 3.5 Flash-Lite text-rate reference projection):**
    *   Single Attempt: `(1,120 * $0.30 / 1M) + (500 * $2.50 / 1M) = $0.001586`
    *   Worst-Case (3 Retries): **$0.004758**
*   **Worst-Case Monthly Cost:** `2,000 * $0.004758 = $9.516`
*   **Break-Even Price:** Not reassessed until image-modality pricing is validated; the current retail price remains unchanged.
*   **Final Retail Price:** **$9.99 / month**
*   **PAYG Gemini 3.8 Flash reference:** Single attempt `(1,120 * $0.75 / 1M) + (500 * $3.75 / 1M) = $0.002715`; three attempts = `$0.008145`. This is a text-rate reference projection, not a confirmed image-modality cost.
*   **Rationale:** The text-rate projections do not confirm multimodal costs or subscription margins. The Pro subscription price remains `$9.99 / month`; validate costs and behavior in Vertex staging before production.

## 5. Data Lifecycle
Upon subscription cancellation, the standard 90-day grace period applies. The user is downgraded to Free tier compute limits but retains full access to export their historical data. On Day 91, a FIFO pruning job truncates their storage to the Free tier limit (100 receipts).