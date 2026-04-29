# Tiering Strategy & FinOps Capacity Planning

This document defines the operational limits, model routing, and break-even pricing analysis for the PDF to Anki (KERA) module.

## 1. Stripe Metadata Contract
Payment links must inject the following custom fields into the Stripe Checkout Session:
*   `pdf_anki_user_id`
*   `language_code`
*   `tier`

## 2. Model Routing per Tier

*   **Free & Admin Tiers:**
    *   **Pipeline:** Legacy Chunked Flow (LlamaParse -> Semantic Chunking -> LanceDB -> Groq `llama-3.3-70b-versatile`).
    *   *Note:* Admin tier shares Pro operational limits but utilizes the Free infrastructure stack.
*   **Premium Tier:**
    *   **Model:** `gemini-2.5-flash-lite` (via Google Vertex AI).
*   **Pro Tier:**
    *   **Model:** `gemini-2.5-flash` (via Google Vertex AI).
*   **PAYG (Pay-As-You-Go):**
    *   **Allowed Models:** Restricted exclusively to Vertex AI (`gemini-2.5-flash-lite`, `gemini-2.5-flash`, `gemini-2.5-pro`).

## 3. Operational Limits

| Metric | Free | Premium | Pro | PAYG (Fair-Use) |
| :--- | :--- | :--- | :--- | :--- |
| **Max Pages per PDF** | 10 | 100 | 500 | N/A |
| **Monthly Page Limit** | 50 | 1,000 | 2,000 | 50,000 |
| **Daily Page Limit** | N/A | N/A | N/A | 2,000 |
| **Max Stored Flashcards** | 100 | 5,000 | 20,000 | 50,000 |

## 4. FinOps & Break-Even Pricing Analysis

*Source of Truth: [Google Cloud Vertex AI Pricing](https://cloud.google.com/vertex-ai/generative-ai/pricing)*

**Current Provider Costs (per 1M tokens):**
*   **Gemini 2.5 Flash Lite:** Input `$0.10`, Output `$0.40`.
*   **Gemini 2.5 Flash:** Input `$0.30`, Output `$2.50`.

**Conservative Baseline Assumptions:**
Calculations utilize the hard limits enforced by `estimate_unified_pdf_token_budget()`:
*   Max input per request: `602,000` tokens.
*   Premium max output: `500 + (100 * 250) = 25,500` tokens.
*   Pro max output: `500 + (300 * 250) = 75,500` tokens.

### Premium Tier Analysis
*   **Max Cost per PDF:**
    *   Input: `602,000 * ($0.10 / 1M) = $0.0602`
    *   Output: `25,500 * ($0.40 / 1M) = $0.0102`
    *   Total: **$0.0704**
*   **Worst-Case Monthly Cost:** (1,000 pages / 100 pages per PDF = 10 PDFs max)
    *   `10 * $0.0704 = $0.704`
*   **Recommended Break-Even Price:** `$0.99 / month`
*   **Final Retail Price:** **$2.99 / month**
*   **Rationale:** Flash-Lite provides exceptional multimodal capabilities at a fraction of the cost, making Premium the tier with the highest relative profit margin.

### Pro Tier Analysis (Model Migration Impact)
The Pro tier was recently migrated from `anthropic/claude-haiku-4.5` to `gemini/gemini-2.5-flash`.

*   **Previous Cost (Claude Haiku 4.5):**
    *   Total per PDF: `$0.9795`
    *   Worst-Case Monthly Cost (4 PDFs): `$3.918`
    *   *Previous Break-Even Price: ~$4.99 / month*
*   **Current Cost (Gemini 2.5 Flash):**
    *   Input: `602,000 * ($0.30 / 1M) = $0.1806`
    *   Output: `75,500 * ($2.50 / 1M) = $0.18875`
    *   Total per PDF: **$0.36935**
*   **Worst-Case Monthly Cost:** (2,000 pages / 500 pages per PDF = 4 PDFs max)
    *   `4 * $0.36935 = $1.4774`
*   **Recommended Break-Even Price:** `$1.99 / month`
*   **Final Retail Price:** **$4.99 / month**
*   **Rationale:** Migrating to Gemini 2.5 Flash reduced the worst-case monthly cost by 62% (from `$3.918` to `$1.4774`). This allows us to maintain the `$4.99` price point while significantly increasing our profit margins, without sacrificing multimodal accuracy.

## 5. PAYG Formula & Privacy
*   **Formula:** `(Exact LLM Provider Cost) * 1.40` (40% Markup).
*   **Privacy Guarantee:** Paid tiers utilize the unified multimodal flow exclusively on Vertex AI. No data is routed through Anthropic or OpenAI, ensuring Zero Data Retention compliance.