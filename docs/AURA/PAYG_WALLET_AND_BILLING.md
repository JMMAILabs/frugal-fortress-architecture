# Pay-As-You-Go (PAYG) Wallet & Dynamic Billing

## 1. FinOps Philosophy
To support high-volume Enterprise users without exposing the platform to runaway LLM costs, the `audio_notes` module implements a prepaid **PAYG Wallet**. 

Instead of hardcoded limits, the system dynamically calculates the maximum allowed audio duration based on the user's real-time USD balance and the exact cost of the selected AI model.

## 2. Dynamic Model Selection (`/llms`)
PAYG users can dynamically switch their summarization model directly from Telegram using the `/llms` command.
* The backend fetches the available models and their costs from the `PricingRegistry` and `DynamicConfigService`.
* When a user selects a model (e.g., `gemini-2.5-pro`), the system calculates: `Max Minutes = Wallet Balance / (Cost Per Minute * 1.40 Markup)`.
* This empowers the user to trade off between cost and intelligence on the fly.

## 3. The Two-Phase Deduction Lifecycle
To prevent Denial of Wallet (DoW) attacks and race conditions, billing is handled transactionally:
1. **Pre-flight Check:** Before processing, the system checks if the user's balance can afford the requested audio duration.
2. **Inference:** The audio is processed via Vertex AI or Groq.
3. **Exact Deduction:** The `LLMResponse` returns the *exact* tokens used. The backend calculates the final USD cost, applies the 40% markup, and deducts it from the PostgreSQL `users` table using a `SELECT ... FOR UPDATE` row-level lock.
4. **Audit Trail:** A `WalletTransaction` record is inserted, providing an immutable ledger of credits and debits.

## 4. Stripe Webhook Integration
When a user tops up their balance via the Stripe Customer Portal:
1. The `checkout.session.completed` webhook is intercepted.
2. The payload signature is cryptographically verified.
3. The USD amount is converted to a Decimal, added to the user's `balance_usd`, and a `CREDIT` transaction is logged.
4. The user receives an automated Telegram message confirming the top-up and their new maximum audio limits.