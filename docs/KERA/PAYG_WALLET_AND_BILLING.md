# Pay-As-You-Go (PAYG) Wallet & Dynamic Billing

## 1. FinOps Philosophy
Processing massive corporate documents (e.g., 500-page PDFs) can easily trigger "Denial of Wallet" scenarios if users select expensive models like `gemini-2.5-pro`. 
To support high-volume Enterprise users safely, the `pdf_anki` module implements a prepaid **PAYG Wallet** with a strict Two-Phase Commit billing architecture.

## 2. Dynamic Model Selection & Fallback
PAYG users can dynamically switch their unified multimodal model directly from the Next.js frontend.
* The backend fetches the available models and their costs from the `PricingRegistry`.
* **Graceful Degradation:** If a user selects an expensive model but their wallet balance is insufficient to cover the *estimated* cost of the PDF, the system does not crash with a 500 error. Instead, it intercepts the request, logs a `pdf_anki_payg_balance_insufficient_fallback` event, and gracefully degrades the processing pipeline to the user's base tier (e.g., the Free LlamaParse + Groq stack).

## 3. The Two-Phase Deduction Lifecycle (2PC)
To prevent race conditions during concurrent PDF uploads, billing is handled transactionally:
1. **Phase 1 (Pre-flight Estimate):** Before processing, `estimate_unified_pdf_token_budget` calculates the maximum possible tokens based on the PDF byte size and the requested flashcard limit. The `BudgetManager` reserves this USD amount in Redis.
2. **Phase 2 (Inference):** The PDF is processed via Vertex AI (Unified Pipeline) or Groq (Chunked Pipeline).
3. **Phase 3 (Exact Deduction):** The `LLMResponse` returns the *exact* tokens used. The backend calculates the final USD cost, applies the 40% markup (`PDF_ANKI_WALLET_CHARGE_MULTIPLIER`), and deducts it from the PostgreSQL `pdf_anki_users` table using a `SELECT ... FOR UPDATE` row-level lock.
4. **Audit Trail:** A `WalletTransaction` record is inserted, providing an immutable ledger of credits and debits.

## 4. Stripe Webhook Integration
When a user tops up their balance via the Stripe Customer Portal:
1. The `checkout.session.completed` webhook is intercepted.
2. The payload signature is cryptographically verified.
3. The USD amount is converted to a Decimal, added to the user's `balance_usd`, and a `CREDIT` transaction is logged.
4. The user's `payg` flag is set to `True`, instantly unlocking the premium model selector in the UI.