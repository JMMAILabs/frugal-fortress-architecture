# Business Rules, Tiering & Enterprise Quotas

This document outlines the operational limits and FinOps constraints currently shipped for the Audio Notes (AURA) module. The forward-looking B2B Enterprise (Shared Quota) design lives in a separate roadmap document — see [AURA Enterprise Roadmap (B2B Shared Quota)](roadmap_enterprise_b2b.md).

> 💲 **Pricing source-of-truth:** [`src/app/core/pricing_registry.py`](https://github.com/JMMAILabs/frugal-fortress-architecture) — last verified 2026-04. The values below are mirrors of that registry; if they diverge, the registry wins.

## 1. Standard Subscription Tiers

### Free Tier (Growth & Acquisition)
*   **Compute Stack:** Groq Whisper-v3 + Llama 3.3 70B.
*   **Max Audio Duration:** 5 minutes per file.
*   **Monthly Quota:** 150 minutes.
*   **History Retention:** Last 20 notes (FIFO eviction).
*   **Active Glossary:** Up to 200 rules. Max 3 glossary exports per day.
*   **Daily Corrections Quota:** 10 operations/day. This shared budget applies to adding new rules or toggling existing rules via `/r`.
*   **Trash Retention:** 30 days.
*   **Price:** $0.00

### Premium Tier
*   **Compute Stack:** Google Vertex AI (Gemini 2.5 Flash Lite) - Single-pass multimodal.
*   **Max Audio Duration:** 20 minutes per file.
*   **Monthly Quota:** 3,000 minutes (50 hours).
*   **History Retention:** Up to 2,000 notes (FIFO eviction).
*   **Active Glossary:** Up to 300 rules. Max 10 glossary exports per day.
*   **Daily Corrections Quota:** 100 operations/day.
*   **Trash Retention:** 60 days + Manual support.
*   **Price:** $4.99 / month

### Pro Tier
*   **Compute Stack:** Google Vertex AI (Gemini 2.5 Flash Lite) - Single-pass multimodal.
*   **Max Audio Duration:** 45 minutes per file.
*   **Monthly Quota:** 9,000 minutes (150 hours).
*   **History Retention:** Up to 2,000 notes (FIFO eviction).
*   **Active Glossary:** Up to 300 rules. Max 20 glossary exports per day.
*   **Daily Corrections Quota:** 100 operations/day.
*   **Trash Retention:** 180 days + Priority support.
*   **Price:** $9.99 / month

## 2. Pay-As-You-Go (PAYG) Architecture
*   **Top-ups:** Minimum $5.00 USD per transaction via Stripe.
*   **Circuit Breaker Limit:** 50,000 minutes/month (Hard cap to prevent client-side bugs or abuse).
*   **Dynamic Context Window:** The maximum audio duration is calculated dynamically at runtime based on:
    1. The selected LLM model.
    2. The user's current USD balance.
    3. The FinOps pricing declared in the `PricingRegistry`.
*   **FinOps Formula:** `(Total LLM Provider Cost) * 1.40` (40% Markup).
*   **UX Requirement:** The available balance must be appended to the footer of every bot response, ensuring the user has real-time visibility without leaving Telegram.

## 3. System Performance & Caching
**Hash-Based Summary Cache (Redis):** Upon receiving an audio file, a SHA-256 hash of the payload is computed. If a match exists (`audio_processed:{hash}`), the summary is served directly from Redis, bypassing Whisper and the LLM entirely.
*   **Cache TTLs:** Free: 24 hours | Premium: 3 days | Pro/PAYG: 7 days.

**Pagination Rate Limits (Redis Token Bucket):**
*   Per minute: 30 requests (Global).
*   Per day: Free (50), Premium (200), Pro (500), PAYG (1000).

## 4. Data Lifecycle & Right to be Forgotten
Retaining data for 90 days post-cancellation is the SaaS industry "Gold Standard" (Win-back strategy).
*   **Day 0 (Cancellation):** The subscription status transitions to `canceled`. Data remains intact.
*   **Day 1 to 90 (Grace Period):** The user is downgraded to Free Tier compute limits (150 mins/month, 5 mins/audio). However, their database records (2,000 notes, 300 rules) are preserved. A persistent UI warning alerts the user to export their glossary before Day 90.
*   **Day 91 (Hard Pruning):** An `Arq` cron job detects the expired grace period and executes a massive FIFO truncation, reducing the user's glossary to the 200 most recent active rules (Free Tier limit).

---

## 5. Roadmap

The forward-looking B2B Enterprise (Shared Quota) design has been moved to its own document so this file only describes shipped behaviour. See [AURA Enterprise Roadmap (B2B Shared Quota)](roadmap_enterprise_b2b.md).