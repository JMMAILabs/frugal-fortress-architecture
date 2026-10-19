# FinOps & Capacity Planning: PDF to Anki (KERA)

This document projects the infrastructure and LLM costs for the `pdf_anki` module. Gemini projections below use ADR-0017 text-token reference rates; they are not confirmed multimodal or Vertex invoice costs.

> 💲 **Pricing source-of-truth:** [`src/app/core/pricing_registry.py`](https://github.com/JMMAILabs/frugal-fortress-architecture) — last verified 2026-08. The values below are mirrors of that registry; if they diverge, the registry wins.

## 1. Baseline Assumptions
*   **1 Page** ≈ 500 words ≈ 666 tokens.
*   **Output per Page** ≈ 5 flashcards ≈ 250 tokens.
*   **Target Volume:** 10,000 pages processed per month.
*   **Total Input Tokens:** ~6.66 Million.
*   **Total Output Tokens:** ~2.50 Million.

## 2. LLM Provider Cost Comparison
We utilize dynamic routing based on the user's subscription tier. The table below compares the cost of processing 10,000 pages across our supported models.

| Provider / Model | Tier | Input Cost (per 1M) | Output Cost (per 1M) | Text-rate reference projection (10k Pages) |
| :--- | :--- | :--- | :--- | :--- |
| **Groq** (`openai/gpt-oss-120b`) | Free / Admin | $0.15 | $0.60 | **$2.50** |
| **Vertex AI** (`gemini-3.1-flash-lite`) | Premium / PAYG | $0.25 | $1.50 | **$5.415** |
| **Vertex AI** (`gemini-3.5-flash-lite`) | Pro / PAYG | $0.30 | $2.50 | **$8.248** |
| **Vertex AI** (`gemini-3.8-flash`) | PAYG | $0.75 | $3.75 | **$14.37** |

*Gemini calculations use the stated 6.66M input and 2.50M output token baseline. The ADR-0017 rates are text-token references and do not confirm PDF/image modality billing or project-specific Vertex invoices. Validate current modality pricing, routes, and behavior in Vertex staging before production. See [ADR-0017](../adr/0017-gemini-models-25-migration.md). Groq figures moved to `openai/gpt-oss-120b` when `llama-3.3-70b-versatile` was decommissioned on 2026-08-16; gpt-oss is a reasoning model, so billed output tokens include reasoning tokens (bounded via `GROQ_REASONING_EFFORT=low`).*

## 3. Architectural Savings (The "Frugal" Impact)
Our architecture implements several layers of optimization that drastically reduce the theoretical costs above:

1. **Exact Hash Caching (L1):**
   *   *Mechanism:* `SHA-256` of the PDF bytes.
   *   *Impact:* If a university class of 100 students uploads the same syllabus, we process it **once**. Cost reduction: **99%** for viral documents.
2. **Semantic Deduplication (L2):**
   *   *Mechanism:* LanceDB vector search per chunk.
   *   *Impact:* If a chunk's embedding matches an existing chunk >95%, we skip LLM generation. Saves ~20% of output tokens on repetitive textbooks.
3. **Token Trimming:**
   *   *Mechanism:* `strip_headers_footers()` removes page numbers and repetitive headers before LLM inference.
   *   *Impact:* Saves ~5% of input tokens per document.

## 4. Profit Margin Analysis (Premium Tier)
*   **Premium Plan Price:** $2.99 / month.
*   **Limit:** 1,000 pages / month.
*   **Max LLM Cost (Gemini 3.1 Flash-Lite text-rate reference projection):** $0.25 (Input) + $0.38 (Output) = **$0.63**.
*   **Gross margin (compute-only text-rate reference projection):** **78.9%**.

The subscription price remains `$2.99 / month`. The text-rate projection does not confirm PDF/image modality costs or actual gross margin; verify provider pricing and behavior in Vertex staging before production.