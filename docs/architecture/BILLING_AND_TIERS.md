# Billing, Tiers & FinOps Strategy

This document outlines the SaaS tiering strategy, operational limits, and Pay-As-You-Go (PAYG) FinOps calculations across all modules of the Frugal Fortress.

> 💲 **Pricing source-of-truth:** [`src/app/core/pricing_registry.py`](https://github.com/JMMAILabs/frugal-fortress-architecture) — last verified 2026-08. The figures below are mirrors of that registry; whenever the two diverge, the registry wins.

## 1. Global Tiering Philosophy
 
The system enforces a strict separation between Free and Paid compute stacks to protect profit margins and ensure enterprise-grade privacy.

*   **Free Tier:** Utilizes third-party APIs (e.g., LlamaParse, Groq GPT-OSS 120B; see [ADR-0015](../adr/0015-groq-gpt-oss-migration.md)). Data may be subject to standard API retention policies. Strictly rate-limited.
*   **Paid Tiers (Premium, Pro, PAYG):** Exclusively utilize Google Vertex AI. Current model mapping: Gemini 3.1 Flash-Lite, Gemini 3.5 Flash-Lite, and Gemini 3.8 Flash; see [ADR-0017](../adr/0017-gemini-models-25-migration.md). Guarantees **Zero Data Retention** (SOC2/GDPR compliant).

## 2. Module-Specific Limits & Margins

### A. VERA (Receipt Parser)
*   **Free:** 30 receipts/month. Max 100 stored. (Stack: LlamaParse + Groq).
*   **Premium ($2.99/mo):** 500 receipts/month. Max 2,000 stored. (Stack: Vertex Flash-Lite).
    *   *FinOps reference projection:* Using ADR-0017 text-token rates and the existing 1,120-input/500-output token budget, one attempt is $0.00103; three attempts are $0.00309 and 500 worst-case receipts project to $1.55. **This is not a confirmed image-modality or Vertex invoice cost.**
*   **Pro ($9.99/mo):** 2,000 receipts/month. Max 5,000 stored. (Stack: Vertex Flash).
    *   *FinOps reference projection:* Using ADR-0017 text-token rates and the existing 1,120-input/500-output token budget, one attempt is $0.001586; three attempts are $0.004758 and 2,000 worst-case receipts project to $9.52. **This is not a confirmed image-modality or Vertex invoice cost.**

### B. KERA (PDF to Anki)
*   **Free:** 50 pages/month. Max 10 pages/PDF. Max 100 flashcards.
*   **Premium ($2.99/mo):** 1,000 pages/month. Max 100 pages/PDF. Max 5,000 flashcards.
    *   *FinOps reference projection:* Using ADR-0017 text-token rates and the existing PDF token budgets, max cost per 100-page PDF = $0.18875; max monthly cost (10 PDFs) = $1.89. **This is not a confirmed multimodal or Vertex invoice cost.**
*   **Pro ($4.99/mo):** 2,000 pages/month. Max 500 pages/PDF. Max 20,000 flashcards.
    *   *FinOps reference projection:* Using ADR-0017 text-token rates and the existing PDF token budgets, max cost per 500-page PDF = $0.36935; max monthly cost (4 PDFs) = $1.48. **This is not a confirmed multimodal or Vertex invoice cost.**

### C. AURA (Audio Notes)
*   **Free:** 150 mins/month. Max 5 mins/audio. (Stack: Groq Whisper + GPT-OSS 120B).
*   **Premium ($4.99/mo):** 3,000 mins/month. Max 20 mins/audio. (Stack: Vertex Flash-Lite Multimodal).
*   **Pro ($9.99/mo):** 9,000 mins/month. Max 45 mins/audio. (Stack: Vertex Flash-Lite Multimodal).

## 3. Pay-As-You-Go (PAYG) Wallet Architecture

For enterprise users exceeding Pro limits, we implement a prepaid wallet system.
*   **Formula:** `(Raw LLM Provider Cost) * 1.40` (40% Markup).
*   **Mechanism:** The `BudgetManager` uses atomic Lua scripts in Redis to reserve funds before inference and commit the exact cost post-inference.
*   **Model Selection:** PAYG users can dynamically select their preferred Vertex AI model (e.g., upgrading from Flash to Pro for complex PDFs). The system dynamically calculates the maximum allowed context window based on their current USD balance.

> **Production validation:** ADR-0017 reference rates are not confirmed project-specific Vertex invoice rates, and its audio rates are estimates. Validate current per-modality pricing, model routes, and behavior in Vertex staging before production; do not treat these provider references as customer prices.

## 4. Data Lifecycle & Churn (Right to be Forgotten)
When a user cancels their subscription via Stripe:
1.  **Grace Period (Days 1-90):** The user is downgraded to Free Tier compute limits, but their historical data (flashcards, receipts, glossary rules) remains intact and exportable.
2.  **Hard Pruning (Day 91):** An automated `Arq` cron job executes a FIFO truncation, permanently deleting vector embeddings and database rows that exceed the Free Tier storage limits.