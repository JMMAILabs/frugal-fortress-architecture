# FinOps & Capacity Planning: PDF to Anki

This document projects the infrastructure and LLM costs for the `pdf_anki` module, demonstrating the economic viability of the "Frugal Fortress" architecture.

## 1. Baseline Assumptions
*   **1 Page** ≈ 500 words ≈ 666 tokens.
*   **Output per Page** ≈ 5 flashcards ≈ 250 tokens.
*   **Target Volume:** 10,000 pages processed per month.
*   **Total Input Tokens:** ~6.66 Million.
*   **Total Output Tokens:** ~2.50 Million.

## 2. LLM Provider Cost Comparison
We utilize dynamic routing based on the user's subscription tier. The table below compares the cost of processing 10,000 pages across our supported models.

| Provider / Model | Tier | Input Cost (per 1M) | Output Cost (per 1M) | Total Cost (10k Pages) |
| :--- | :--- | :--- | :--- | :--- |
| **Groq** (`llama-3.3-70b`) | Free / Admin | $0.59 | $0.79 | **$5.90** |
| **Vertex AI** (`gemini-2.5-flash-lite`) | Premium / PAYG | $0.10 | $0.40 | **$1.66** |
| **Vertex AI** (`gemini-2.5-flash`) | Pro / PAYG | $0.30 | $2.50 | **$8.25** |

*Note: Prices based on official provider pricing as of April 2026, reflected in `src/app/core/pricing_registry.py`.*

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
*   **Max LLM Cost (Gemini Flash-Lite):** $0.16 (Input) + $0.10 (Output) = **$0.26**.
*   **Gross Margin (Compute only):** **91.3%**.

By leveraging Vertex AI's Flash-Lite model for the Premium tier, we maintain multimodal capabilities (processing images/tables in PDFs) while preserving SaaS-level profit margins.