# Security & Compliance Architecture (SOC2 / GDPR / HIPAA)

This document outlines the data lifecycle, encryption standards, and privacy guardrails implemented in the Frugal Fortress architecture to meet strict Enterprise compliance requirements.

## 1. Data Lifecycle & Zero Data Retention
We operate under a strict **Zero-Trust** and **Data Minimization** policy.

1. **Ingestion:** All payloads (PDFs, Images, Audio) are transmitted via HTTPS (TLS 1.3).
2. **Ephemeral Storage:** Raw files are encrypted and stored in Redis with a strict **1-hour TTL**. They are purged immediately after the background worker (`Arq`) processes them.
3. **Inference Processing:**
   *   **Paid Tiers (Premium, Pro, PAYG):** Processed exclusively via Google Vertex AI. We rely on Google Cloud's Enterprise terms, which guarantee **Zero Data Retention** (customer data is NEVER used to train foundational models).
   *   **Free Tier:** Processed via Groq/LlamaParse. Users explicitly consent to standard API terms.
4. **Right to be Forgotten (Pruning):** If a user cancels their subscription, a 90-day grace period begins. On day 91, an automated cron job (`maintain_pdf_data_lifecycle`) permanently deletes all associated vector embeddings, flashcards, and receipts.

## 2. Encryption Standards
### A. Encryption in Transit
All internal (VPC) and external communications are secured via TLS 1.2/1.3.

### B. Application-Level Encryption (ALE)
To protect against database compromises (e.g., a DBA accessing raw tables or a SQL injection), sensitive user data is encrypted *before* it leaves the Python application.
*   **Implementation:** `cryptography.fernet` (AES-128-CBC with HMAC-SHA256).
*   **Target Data:** Glossary Rules, Transcription logs, Flashcard Fronts/Backs, Deck Titles, Receipt Payloads, and User Profile data (Email, Display Name).
*   **Reference:** `src/app/modules/audio_notes/infrastructure/audio_notes_ale.py` and `src/app/security/feedback_privacy.py`.

## 3. Data Loss Prevention (DLP) & PII Scrubbing
Before any text is sent to an external LLM (especially on the Free Tier), it passes through our internal scrubbing pipeline.
*   **Regex Pass:** Deterministic removal of Emails, Phone Numbers, SSNs, and Credit Cards.
*   **NLP Pass (Microsoft Presidio):** Contextual Named Entity Recognition (NER) replaces names and organizations with safe tokens (e.g., `[PERSON]`, `[ORG]`).
*   **Fail-Secure Local SLM (Optional):** A local Llama-3.2-3B model evaluates prompts for complex PHI/PII leaks. If a leak is detected, or if the local model crashes, the request is blocked (`SecurityPolicyViolation`). It fails closed.

## 4. Access Control & Multi-Tenancy
*   **Authentication:** Handled via Google OAuth2 (OpenID Connect).
*   **Module Isolation:** JWTs are strictly scoped per module (`module=pdf_anki` vs `module=receipt_parser`) to prevent lateral movement between product contexts.
*   **Tenant Isolation (Stateful Stores):** The `ContextMiddleware` binds the `tenant_id` to Python `contextvars`. All database repositories and vector searches (LanceDB/pgvector) enforce a hard `WHERE tenant_id = :tenant_id` clause. Cross-tenant data leakage is mathematically impossible at the query level for stateful stores.
*   **Content-Addressable Idempotency Caches:** Hash-keyed deduplication caches (`audio_processed:{sha256}`, `pdf:deck:{file_hash}:{sig}`) are intentionally **content-addressable, not tenant-scoped**, so platform-wide viral content is processed once. These caches store **only tenant-agnostic LLM output** (no per-user RAG/glossary context is mixed in) and are auditable via `FinOpsCallback` logs. See [ADR-0013](../adr/0013-sha256-idempotency-guard.md) and the explicit invariant exception in [CORE_INVARIANTS §2.2](CORE_INVARIANTS.md).