# docs/audio_notes/finops_capacity_planning.md

# FinOps & Capacity Planning: Audio Notes

This document projects the infrastructure and LLM costs for the `audio_notes` module, demonstrating the economic viability of the "Frugal Fortress" architecture.

## 1. Baseline Assumptions
*   **1 Minute of Audio** ≈ 150 words ≈ 200 tokens (Transcript) + 512 tokens (Summary).
*   **Target Volume:** 10,000 minutes (166 hours) processed per month.

## 2. LLM Provider Cost Comparison (Per Minute of Audio)

We utilize dynamic routing based on the user's subscription tier.

| Provider / Model | Tier | Audio/Whisper Cost | Text Output Cost | Total Cost (per min) | Total Cost (10k mins) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Groq** (`whisper-large-v3` + `llama-3.3`) | Free | $0.0018 | $0.0004 | **$0.0022** | **$22.00** |
| **Vertex AI** (`gemini-2.5-flash-lite`) | Premium | $0.0000 (Included in multimodal) | $0.0002 | **$0.0002** | **$2.00** |
| **Vertex AI** (`gemini-2.5-flash`) | Pro | $0.0000 (Included in multimodal) | $0.0012 | **$0.0012** | **$12.00** |

*Note: Vertex AI processes audio natively in the multimodal prompt, eliminating the need for a separate Whisper transcription step, drastically reducing costs for Premium/Pro tiers.*

## 3. Architectural Savings (The "Frugal" Impact)
1. **SHA-256 Idempotency Guard:** If a viral audio (e.g., a forwarded WhatsApp/Telegram voice note) is sent by multiple users, the Redis cache intercepts the hash. Cost reduction: **100%** for duplicate audios. Latency drops from ~4s to <50ms.