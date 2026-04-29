# VERA: Verified Expense & Receipt Architecture

**VERA** is the Cost-Efficient Financial Extraction engine of the Frugal Fortress. It is a hybrid AI architecture designed to extract structured financial data from receipts and invoices with maximum privacy and minimum cost.

By leveraging Edge computing to reduce payload sizes and Cloud orchestration for intelligent extraction, VERA solves the traditional bottlenecks of Vision-Language Models (VLMs).

## Core Capabilities

*   **Edge Compute Offloading:** The Next.js PWA captures the image, resizes it via the HTML5 Canvas API, and exports a highly compressed JPEG. This reduces payload sizes from ~5MB to ~150KB, slashing bandwidth and VLM token consumption by over 90% before the request even hits the backend.
*   **Deterministic Math Validation & Self-Correction:** LLMs hallucinate math. VERA uses a pure Python `MathValidator` to ensure `Subtotal - Discount + Tax + Shipping = Total`. If validation fails, the exact error is injected back into the LLM prompt to force a self-correction loop (up to 3 retries).
*   **Human-in-the-Loop (HITL) via HTTP 206:** If the AI fails to balance the math after 3 retries, the API does not throw a 500 error. It returns an `HTTP 206 Partial Content` with the AI's "best effort" JSON, allowing the user to correct the math in the UI.
*   **Few-Shot Vendor Matching (RAG):** When a receipt is validated, the `issuer_name` is embedded and stored in PostgreSQL via `pgvector`. On subsequent uploads, the system performs a cosine similarity search to find previously validated receipts from the same vendor, injecting them as `<example>` blocks to adapt to obscure layouts without fine-tuning.
*   **Application-Level Encryption (ALE):** Financial totals and full JSON payloads are encrypted at rest using AES-128-CBC (Fernet). Data is decrypted on-the-fly strictly in memory during CSV/XLSX exports.

## Deep Dives

Explore the specific architectural implementations for this module:

* [FinOps & Capacity Planning](finops_capacity_planning.md)
* [Tiers, Limits & Stripe Integration](tiers.md)
*[Troubleshooting & LlamaParse DNS](troubleshooting.md)