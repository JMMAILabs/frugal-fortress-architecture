# ADR 0013: Global SHA-256 Idempotency Guard for Viral Audio
**Date:** 2026-05-12
**Status:** Accepted

## Context
In messaging platforms like Telegram, users frequently forward "viral" or popular voice notes (e.g., a CEO's company-wide address, a popular podcast clip).
Processing the exact same audio file multiple times through Whisper (Transcription) and Llama-3 (Summarization) wastes expensive LLM compute and introduces unnecessary latency.

## Decision
We implemented a strict, **Content-Addressable Idempotency Guard** at the edge of the application layer, backed by Redis.
1. **Hashing:** Upon receiving an audio file, the backend immediately calculates the `SHA-256` hash of the raw audio bytes.
2. **Cache Lookup:** It checks Redis for the key `audio_processed:{hash}`. The key is intentionally **content-addressable** (it does **not** include `tenant_id`), so byte-identical viral content is deduplicated platform-wide.
3. **Short-Circuit:** If a match is found, the backend instantly returns the cached `summary_text` and `transcript` to the new user. The LLM providers are never invoked.
4. **TTL:** The cache Time-To-Live (TTL) is tiered (e.g., 24 hours for Free users, 7 days for Pro users).

## Tenant-Isolation Caveat
This cache is the **explicit exception** to the "every cache key MUST include tenant_id" rule defined in [CORE_INVARIANTS §2.2](../architecture/CORE_INVARIANTS.md). The exception is bounded by two preconditions that MUST always hold for any cache that adopts this pattern:
1. The cached value is **byte-identical, tenant-agnostic LLM output** — it does NOT embed any per-user RAG retrieval, glossary substitution, or other personalized context at the moment of write.
2. The cache stores only the post-LLM summary/transcript — never raw user payloads or PII.

Any future feature that needs to mix tenant-specific context into the cached output MUST either (a) include `tenant_id` in the key, or (b) cache before the personalization step. See also [security_compliance_soc2 §4](../architecture/security_compliance_soc2.md) and [MULTI_TENANCY_AND_ISOLATION §3](../architecture/MULTI_TENANCY_AND_ISOLATION.md).

## Consequences
* **Positive:** Reduces latency for duplicate audio from ~4 seconds to <50 milliseconds.
* **Positive:** Saves 100% of the LLM API costs for viral media across the entire platform.
* **Negative:** Requires holding the audio bytes in memory briefly to compute the hash, which is acceptable given our 25MB file size limit.
* **Operational:** Code reviews of changes to `process_audio` / `process_pdf_task` MUST verify the two preconditions above before approving any modification to the cache write path.