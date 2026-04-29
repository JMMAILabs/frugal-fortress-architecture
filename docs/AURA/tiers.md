# Business Rules, Tiering & Enterprise Quotas

This document outlines the operational limits, FinOps constraints, and the architectural roadmap for the B2B Enterprise tier within the Audio Notes module.

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

## 5. Architectural Implementation Plan: Enterprise Tier (B2B Shared Quota)

To support B2B Enterprise clients, we will implement a "Shared Quota" architecture. This allows an Enterprise Admin to share their subscription limits and custom AI models with their employees.

### Phase 1: Domain Evolution (Self-Referential Accounts)
Instead of creating complex `Team` tables, we will utilize a self-referential relationship on the `User` entity.
*   **Database Schema:** Add `parent_account_id` (String, nullable, indexed, ForeignKey to `users.telegram_id`).
    *   Admin: `parent_account_id` is `NULL`.
    *   Employee: `parent_account_id` points to the Admin's `telegram_id`.
*   **Domain Logic (`User` Entity):**
    *   `get_effective_tier()`: Returns the parent's tier if linked, otherwise its own.
    *   `get_effective_tenant_id()`: Returns `parent_account_id` if linked, otherwise `telegram_id`. **Critical for Redis FinOps tracking.**

### Phase 2: Consumption Routing & FinOps
*   **Token Buckets:** When interacting with the `BudgetManager` or `RateLimitMiddleware`, the system MUST use the `effective_tenant_id`. This mathematically guarantees that all employees draw from the Admin's single Redis token bucket. No race conditions.
*   **Shared Corporate Memory:** The `get_glossary_context` use case must fetch rules using the `effective_tenant_id`. If the Admin corrects a technical term, the AI learns it for all employees instantly.

### Phase 3: Stripe Adapter & Dynamic Quoting
*   **Entity Overrides:** Add `custom_model`, `custom_max_duration_min`, and `custom_monthly_minutes` to the `User` entity to support negotiated Enterprise contracts without hardcoding `if/else` logic.
*   **Payment Gateway Port:** Implement `create_dynamic_checkout` using the Stripe SDK.
*   **Stateless Provisioning:** Inject a `metadata` dictionary into the Stripe Checkout Session containing the custom limits. The `POST /stripe/webhook` endpoint will extract this metadata upon `checkout.session.completed` and provision the Enterprise limits transactionally.