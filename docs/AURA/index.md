# Module: AURA (Audio Understanding & Retention Architecture)

## 1. Architectural Overview
**AURA** is the Corporate Memory Engine of the Frugal Fortress. It transforms unstructured voice messages into structured, actionable corporate memory via a direct Telegram Webhook integration.

Unlike naive wrappers around OpenAI, AURA processes audio asynchronously while maintaining strict idempotency, FinOps controls, and SOC2 compliance.

**Core Flow:**
1. **Ingestion:** Telegram Webhook receives the audio file.
2. **Idempotency Guard:** The system calculates the `SHA-256` hash of the audio bytes. If a viral/forwarded audio is detected, it serves the summary directly from Redis (<50ms), bypassing all LLM compute.
3. **Dynamic Routing:**
   * *Free Tier:* Audio is transcribed via Groq (`whisper-large-v3`) and summarized via Groq (`llama-3.3-70b-versatile`).
   * *Paid Tiers (Premium/Pro/PAYG):* Audio is sent directly to Google Vertex AI (`gemini-2.5-flash-lite` / `gemini-2.5-flash`/ `gemini-2.5-pro`) for single-pass multimodal extraction.
4. **Delivery:** The structured Markdown summary is sent back to the user via the Telegram API.

## 2. Key Architectural Patterns
This module heavily leverages the Frugal Fortress core patterns. For detailed technical decisions, refer to the following Architecture Decision Records (ADRs):

* **[ADR-0011: Redis-backed Media Group Debouncing](../adr/0011-redis-media-group-debouncing.md):** How we handle concurrent Telegram image uploads for Support Tickets without race conditions.
* **[ADR-0012: Two-Phase Commit for PAYG Wallet](../adr/0012-two-phase-commit-payg.md):** How we prevent Denial of Wallet (DoW) attacks using atomic Lua scripts in Redis.
* **[ADR-0013: SHA-256 Idempotency Guard](../adr/0013-sha256-idempotency-guard.md):** How we achieve 100% cost savings on viral/duplicate audio messages.
* **[ADR-0014: Dynamic RAG Injection (Nested Learning Loop) ](../adr/0014-dynamic-rag-injection.md):** How the system learns user-specific jargon in real-time without expensive model fine-tuning.

## 3. Security & Privacy (SOC2)
* **Zero Data Retention:** Paid tiers utilize Vertex AI under enterprise agreements; audio data is never used for model training.
* **Application-Level Encryption (ALE):** User data, glossary rules and transcription logs are encrypted at rest using AES-128-CBC (Fernet). Even with direct database access, the raw transcripts remain secure.
* **Data Lifecycle:** Canceled subscriptions enter a 90-day grace period before a background cron job (`Arq`) permanently prunes their historical data.

## 4. Documentation & Operations
* 💰 **[FinOps & PAYG Wallet](PAYG_WALLET_AND_BILLING.md):** Details on Stripe webhooks and dynamic model pricing.
* 🎫 **[Support Tickets Architecture](SUPPORT_TICKETS_ARCHITECTURE.md):** Details on the native Telegram Helpdesk integration.
* 📈 **[Capacity Planning](finops_capacity_planning.md):** Cost matrix and ROI projections.
* 📊 **[Tiers & Limits](tiers.md):** Business rules for Free, Premium, Pro, and PAYG users.

## 5. Diagrams
*[C4 Model (Context, Container, Component)](../diagrams/AURA/c4_model.md)
*[Audio Processing & Learning Flow](../diagrams/AURA/diagram_flow.md)
* [PAYG Billing Lifecycle](../diagrams/AURA/diagram_payg_billing.md)
* [Support Tickets & Debouncing](../diagrams/AURA/diagram_support_tickets.md)
* [Glossary Management](../diagrams/AURA/diagram_glossary_management.md)