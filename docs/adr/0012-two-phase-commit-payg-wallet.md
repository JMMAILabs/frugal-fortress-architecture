# ADR 0012: Two-Phase Commit for PAYG Wallet Deductions

**Date:** 2026-05-12
**Status:** Accepted

## Context
The Audio Notes module supports a Pay-As-You-Go (PAYG) tier where users prepay a USD balance via Stripe. Users can dynamically select expensive models (e.g., Gemini 1.5 Pro). 
We must ensure that users cannot exploit concurrent requests to bypass their wallet balance (Denial of Wallet attack), while also ensuring we only charge them for the *exact* tokens consumed during inference.

## Decision
We implemented a Two-Phase Commit (2PC) billing pattern using Redis Lua scripts and PostgreSQL row-level locks.

1. **Phase 1 (Reserve):** Before calling the LLM, the `BudgetManager` executes an atomic Lua script in Redis to reserve the *estimated* maximum cost of the audio transcription and summarization. If `current_spend + reserved > balance`, the request is rejected (HTTP 402/429).
2. **Phase 2 (Commit/Deduct):** After the LLM returns the exact `prompt_tokens` and `completion_tokens`, the backend calculates the precise USD cost (including our 40% markup). 
3. **Persistence:** The backend executes a `SELECT ... FOR UPDATE` on the PostgreSQL `users` table to safely deduct the exact amount from `balance_usd` and inserts an immutable `WalletTransaction` record. Finally, the Redis reservation is cleared.

## Consequences
* **Positive:** Mathematically guarantees that a user cannot spend more than their balance, even if they send 100 concurrent audio messages.
* **Positive:** Users are billed fairly for exact usage, not estimates.
* **Negative:** Requires strict `try/finally` blocks to ensure Redis reservations are released if the LLM provider times out.