# FinOps & Capacity Planning: Audio Notes (AURA)

> 💲 **Pricing source-of-truth:** [`src/app/core/pricing_registry.py`](https://github.com/JMMAILabs/frugal-fortress-architecture) — last verified 2026-08. The values below are mirrors of that registry; if they diverge, the registry wins.

This document projects the infrastructure and LLM costs for the `audio_notes` module, demonstrating the economic viability of the "Frugal Fortress" architecture.

## 1. Baseline Assumptions
*   **1 Minute of Audio** ≈ 150 words; retain 512 output text tokens per summary for this reference projection. The call sends audio natively, so transcript text tokens are not counted as a separate input charge.
*   **Target Volume:** 10,000 minutes (166 hours) processed per month.

## 2. LLM Provider Cost Comparison (Per Minute of Audio)

We utilize dynamic routing based on the user's subscription tier.

| Provider / Model | Tier | Audio billing reference (unit shown) | Text output reference (512 tokens/min) | Text output (10k mins) |
| :--- | :--- | :--- | :--- | :--- |
| **Groq** (`whisper-large-v3` + `gpt-oss-120b`) | Free | Existing estimate: $0.0018/min | Existing estimate: $0.0003/min | **$3.00** |
| **Vertex AI** (`gemini-3.1-flash-lite`) | Included Premium / Pro | **$0.50 / 1M audio tokens (estimate; not confirmed Vertex billing)** | $0.000768/min | **$7.68** |
| **Vertex AI** (`gemini-3.5-flash-lite`) | PAYG | **$0.30 / 1M audio tokens (estimate; not confirmed Vertex billing)** | $0.001280/min | **$12.80** |
| **Vertex AI** (`gemini-3.8-flash`) | PAYG | **$0.75 / 1M audio tokens (estimate; not confirmed Vertex billing)** | $0.001920/min | **$19.20** |

*The Vertex output figures are text-token reference projections only, calculated using ADR-0017 rates ($1.50/$2.50/$3.75 per million output tokens). The separate audio rates from ADR-0017 are estimates, not confirmed Vertex billing. Total audio cost cannot be calculated without validated audio-token usage and modality pricing; the text figures above are not total per-minute costs. Validate prices, model routes, and behavior in Vertex staging before production. See [ADR-0017](../adr/0017-gemini-models-25-migration.md).*

## 3. Architectural Savings (The "Frugal" Impact)
1. **SHA-256 Idempotency Guard:** If a viral audio (e.g., a forwarded WhatsApp/Telegram voice note) is sent by multiple users, the Redis cache intercepts the hash. Cost reduction: **100%** for duplicate audios. Latency drops from ~4s to <50ms.