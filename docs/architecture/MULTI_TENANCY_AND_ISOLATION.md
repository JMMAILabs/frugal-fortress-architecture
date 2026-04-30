# docs/architecture/MULTI_TENANCY_AND_ISOLATION.md

# Multi-Tenancy & Data Isolation Strategy
 
## 1. The Enterprise Requirement
In a B2B SaaS environment, the most critical security invariant is **Tenant Isolation**. A bug in the application logic must never result in Tenant A accessing Tenant B's invoices, audio notes, or flashcards.
 
## 2. Context Propagation (The `AppContext`)
We do not rely on developers manually passing `tenant_id` through every function signature (which is error-prone). Instead, we use Python's native `contextvars`. 

1. **Middleware Interception:** The `ContextMiddleware` intercepts every incoming HTTP request. It extracts the authenticated user's ID (from the JWT) and binds it to a global `tenant_id` context variable.
2. **Traceability:** This `tenant_id` is automatically injected into every `structlog` JSON log entry and OpenTelemetry span.
3. **Background Workers:** When a job is enqueued to Arq (e.g., `process_pdf_task`), the `tenant_id` is serialized into the job payload and restored in the worker's context before execution begins.

## 3. Database-Level Isolation
While Application-Level Encryption (ALE) protects data at rest, we enforce logical isolation at the query level.

*   **Strict Repository Pattern:** All database access goes through Repository classes (e.g., `ReceiptRepository`). Every `SELECT`, `UPDATE`, and `DELETE` statement is hardcoded to include a `WHERE tenant_id = :tenant_id` clause.
*   **Vector Search Isolation:** When performing semantic searches in `pgvector` or `LanceDB` (e.g., finding Few-Shot examples for the Receipt Parser), the `tenant_id` is applied as a pre-filter. The LLM will *never* see another company's data as context.
*   **Idempotency Cache Caveat (Out of Scope):** Content-addressable Redis caches keyed by `SHA-256` of the raw input (`audio_processed:{hash}`, `pdf:deck:{file_hash}:{sig}`) are deliberately **not** tenant-scoped — they are a platform-wide deduplication layer for viral/duplicate content (see [ADR-0013](../adr/0013-sha256-idempotency-guard.md)). They do not store any tenant-specific RAG/glossary context, so they fall outside the tenant-isolation invariant.

## 4. FinOps Isolation (Token Buckets)
Tenant isolation extends to infrastructure costs.
*   **Rate Limiting:** Redis keys for rate limiting are namespaced by tenant (`rl:{tenant_id}:rpm`).
*   **Budgeting:** The `BudgetManager` uses atomic Lua scripts to reserve and commit USD costs against a specific `tenant_id`. If a tenant exhausts their PAYG balance, only their requests are halted (HTTP 402/429), protecting the platform's overall profitability.