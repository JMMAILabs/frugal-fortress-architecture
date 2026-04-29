# Billing, Tiers & FinOps Strategy

This document outlines the SaaS tiering strategy, operational limits, and Pay-As-You-Go (PAYG) FinOps calculations across all modules of the Frugal Fortress.

## 1. Global Tiering Philosophy
 
The system enforces a strict separation between Free and Paid compute stacks to protect profit margins and ensure enterprise-grade privacy.

*   **Free Tier:** Utilizes third-party APIs (e.g., LlamaParse, Groq Llama-3.3). Data may be subject to standard API retention policies. Strictly rate-limited.
*   **Paid Tiers (Premium, Pro, PAYG):** Exclusively utilizes Google Vertex AI (Gemini 2.5 Flash/Pro). Guarantees **Zero Data Retention** (SOC2/GDPR compliant).

## 2. Module-Specific Limits & Margins

### A. VERA (Receipt Parser)
*   **Free:** 30 receipts/month. Max 100 stored. (Stack: LlamaParse + Groq).
*   **Premium ($2.99/mo):** 500 receipts/month. Max 2,000 stored. (Stack: Vertex Flash-Lite).
    *   *FinOps:* Max cost per receipt (with 3 retries) = $0.0009. Max monthly cost = $0.46. **Margin: 84%**.
*   **Pro ($9.99/mo):** 2,000 receipts/month. Max 5,000 stored. (Stack: Vertex Flash).
    *   *FinOps:* Max cost per receipt (with 3 retries) = $0.0047. Max monthly cost = $9.51. **Margin: ~5%** (Loss-leader / High-volume tier).

### B. KERA (PDF to Anki)
*   **Free:** 50 pages/month. Max 10 pages/PDF. Max 100 flashcards.
*   **Premium ($2.99/mo):** 1,000 pages/month. Max 100 pages/PDF. Max 5,000 flashcards.
    *   *FinOps:* Max cost per 100-page PDF = $0.07. Max monthly cost (10 PDFs) = $0.70. **Margin: 76%**.
*   **Pro ($4.99/mo):** 2,000 pages/month. Max 500 pages/PDF. Max 20,000 flashcards.
    *   *FinOps:* Max cost per 500-page PDF = $0.36. Max monthly cost (4 PDFs) = $1.47. **Margin: 70%**.

### C. AURA (Audio Notes)
*   **Free:** 150 mins/month. Max 5 mins/audio. (Stack: Groq Whisper + Llama-3.3).
*   **Premium ($4.99/mo):** 3,000 mins/month. Max 20 mins/audio. (Stack: Vertex Flash-Lite Multimodal).
*   **Pro ($9.99/mo):** 9,000 mins/month. Max 45 mins/audio. (Stack: Vertex Flash-Lite Multimodal).

## 3. Pay-As-You-Go (PAYG) Wallet Architecture

For enterprise users exceeding Pro limits, we implement a prepaid wallet system.
*   **Formula:** `(Raw LLM Provider Cost) * 1.40` (40% Markup).
*   **Mechanism:** The `BudgetManager` uses atomic Lua scripts in Redis to reserve funds before inference and commit the exact cost post-inference.
*   **Model Selection:** PAYG users can dynamically select their preferred Vertex AI model (e.g., upgrading from Flash to Pro for complex PDFs). The system dynamically calculates the maximum allowed context window based on their current USD balance.

## 4. Data Lifecycle & Churn (Right to be Forgotten)
When a user cancels their subscription via Stripe:
1.  **Grace Period (Days 1-90):** The user is downgraded to Free Tier compute limits, but their historical data (flashcards, receipts, glossary rules) remains intact and exportable.
2.  **Hard Pruning (Day 91):** An automated `Arq` cron job executes a FIFO truncation, permanently deleting vector embeddings and database rows that exceed the Free Tier storage limits.