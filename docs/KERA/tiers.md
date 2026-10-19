# Tiering Strategy & FinOps Capacity Planning

This document defines the operational limits, model routing, and break-even pricing analysis for the PDF to Anki (KERA) module.

> 💲 **Pricing source-of-truth:** [`src/app/core/pricing_registry.py`](https://github.com/JMMAILabs/frugal-fortress-architecture) — last verified 2026-08. The values below are mirrors of that registry; if they diverge, the registry wins.

## 1. Stripe Metadata Contract
Payment links must inject the following custom fields into the Stripe Checkout Session:
*   `pdf_anki_user_id`
*   `language_code`
*   `tier`

## 2. Model Routing per Tier

*   **Free & Admin Tiers:**
    *   **Pipeline:** Legacy Chunked Flow (LlamaParse -> Semantic Chunking -> LanceDB -> Groq `openai/gpt-oss-120b`).
    *   *Note:* Admin tier shares Pro operational limits but utilizes the Free infrastructure stack.
*   **Premium Tier:**
    *   **Model:** `gemini-3.1-flash-lite` (via Google Vertex AI).
*   **Pro Tier:**
    *   **Model:** `gemini-3.5-flash-lite` (via Google Vertex AI).
*   **PAYG (Pay-As-You-Go):**
    *   **Allowed Models:** Restricted exclusively to Vertex AI (`gemini-3.1-flash-lite`, `gemini-3.5-flash-lite`, `gemini-3.8-flash`).

## 3. Operational Limits

| Metric | Free | Premium | Pro | PAYG (Fair-Use) |
| :--- | :--- | :--- | :--- | :--- |
| **Max Pages per PDF** | 10 | 100 | 500 | N/A |
| **Monthly Page Limit** | 50 | 1,000 | 2,000 | 50,000 |
| **Daily Page Limit** | N/A | N/A | N/A | 2,000 |
| **Max Stored Flashcards** | 100 | 5,000 | 20,000 | 50,000 |

## 4. FinOps & Break-Even Pricing Analysis

*Source of Truth: [Google Cloud Vertex AI Pricing](https://cloud.google.com/vertex-ai/generative-ai/pricing)*

**ADR-0017 text-token reference rates (per 1M tokens; not confirmed project-specific Vertex invoice rates):**
*   **Gemini 3.1 Flash-Lite:** Input `$0.25`, Output `$1.50`.
*   **Gemini 3.5 Flash-Lite:** Input `$0.30`, Output `$2.50`.
*   **Gemini 3.8 Flash:** Input `$0.75`, Output `$3.75`.

These text-rate projections do not establish image/PDF modality billing. Validate current per-modality prices, model routes, and behavior in Vertex staging before production. See [ADR-0017](../adr/0017-gemini-models-25-migration.md).

**Conservative Baseline Assumptions:**
Calculations utilize the hard limits enforced by `estimate_unified_pdf_token_budget()`:
*   Max input per request: `602,000` tokens.
*   Premium max output: `500 + (100 * 250) = 25,500` tokens.
*   Pro max output: `500 + (300 * 250) = 75,500` tokens.

### Premium Tier Analysis
*   **Max Cost per PDF:**
    *   Input: `602,000 * ($0.25 / 1M) = $0.1505`
    *   Output: `25,500 * ($1.50 / 1M) = $0.03825`
    *   Total (text-rate reference projection): **$0.18875**
*   **Worst-Case Monthly Cost:** (1,000 pages / 100 pages per PDF = 10 PDFs max)
    *   `10 * $0.18875 = $1.8875`
*   **Recommended Break-Even Price:** `$0.99 / month`
*   **Final Retail Price:** **$2.99 / month**
*   **Rationale:** The Premium provider-cost projection uses the ADR-0017 text-token reference rates; its actual multimodal costs and margin require Vertex staging and invoice validation. The subscription price remains `$2.99 / month`.

### Pro Tier Analysis

*   **Cost (Gemini 3.5 Flash-Lite):**
    *   Input: `602,000 * ($0.30 / 1M) = $0.1806`
    *   Output: `75,500 * ($2.50 / 1M) = $0.18875`
    *   Total per PDF (text-rate reference projection): **$0.36935**
*   **Worst-Case Monthly Cost:** (2,000 pages / 500 pages per PDF = 4 PDFs max)
    *   `4 * $0.36935 = $1.4774`
*   **Recommended Break-Even Price:** `$1.99 / month`
*   **Final Retail Price:** **$4.99 / month**
*   **Rationale:** The Pro provider-cost projection uses the ADR-0017 text-token reference rates; its actual multimodal costs and margin require Vertex staging and invoice validation. The subscription price remains `$4.99 / month`.

## 5. PAYG Formula & Privacy
*   **Formula:** `(Exact LLM Provider Cost) * 1.40` (40% Markup).
*   **Privacy Guarantee:** Paid tiers utilize the unified multimodal flow exclusively on Google Vertex AI under enterprise terms (Zero Data Retention). The platform does not route any traffic through other LLM vendors.