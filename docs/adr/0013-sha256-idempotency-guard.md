# ADR 0013: Global SHA-256 Idempotency Guard for Viral Audio
**Date:** 2026-05-12
**Status:** Accepted

## Context
In messaging platforms like Telegram, users frequently forward "viral" or popular voice notes (e.g., a CEO's company-wide address, a popular podcast clip).
Processing the exact same audio file multiple times through Whisper (Transcription) and Llama-3 (Summarization) wastes expensive LLM compute and introduces unnecessary latency.

## Decision
We implemented a strict, **Global Idempotency Guard** at the edge of the application layer, backed by Redis.
1. **Hashing:** Upon receiving an audio file, the backend immediately calculates the `SHA-256` hash of the raw audio bytes.
2. **Global Cache Lookup:** It checks Redis for the key `audio_processed:{hash}`. Notice this key does **not** include the `tenant_id`.
3. **Short-Circuit:** If a match is found, the backend instantly returns the cached `summary_text` and `transcript` to the new user. The LLM providers are never invoked.
4. **TTL:** The cache Time-To-Live (TTL) is tiered (e.g., 24 hours for Free users, 7 days for Pro users).

## Consequences
* **Positive:** Reduces latency for duplicate audio from ~4 seconds to <50 milliseconds.
* **Positive:** Saves 100% of the LLM API costs for viral media across the entire platform.
* **Negative:** Requires holding the audio bytes in memory briefly to compute the hash, which is acceptable given our 25MB file size limit.