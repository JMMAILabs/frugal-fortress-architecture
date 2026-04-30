# Database Schema (ERD)

This diagram covers every SQLAlchemy ORM entity registered against `Base` in the backend. Entities are grouped by bounded context. Sensitive columns are encrypted at rest via the Fernet-based ALE pipeline (`src/app/security/encryption.py`); these are marked **🔒** below.

```mermaid
erDiagram
    %% ============================================================
    %% Cross-cutting (shared base / domain)
    %% ============================================================
    FEEDBACK_LOG {
        uuid id PK
        uuid request_id UK "indexed"
        string tenant_id "multi-tenancy"
        text user_prompt
        text llm_response
        vector embedding "pgvector (HNSW)"
        tsvector search_vector "GIN"
        float score
        float cost_usd "FinOps"
        datetime created_at "partition key (RANGE month)"
    }

    WALLET_TRANSACTIONS {
        uuid id PK
        string user_id "indexed"
        string module "audio_notes / pdf_anki / receipt_parser"
        string kind "credit / debit"
        decimal amount_usd
        string idempotency_key UK
        json metadata
        datetime created_at
    }

    PROMPT_TEMPLATES {
        uuid id PK
        string name UK
        string version
        text content "Jinja2"
        json input_variables
        bool is_active
        datetime updated_at
    }

    %% ============================================================
    %% Module: audio_notes (AURA)  — _TABLE_PREFIX = "audio_notes_"
    %% ============================================================
    AUDIO_NOTES_USERS {
        string telegram_id PK
        string tier "🔒 ALE"
        string base_tier "🔒 ALE"
        string balance_usd "🔒 ALE"
        string payg_summary_model "🔒 ALE"
        string preferred_language_code "🔒 ALE"
        datetime first_seen "🔒 ALE"
        datetime last_active "🔒 ALE"
        int daily_audio_count "🔒 ALE"
        int daily_correction_count "🔒 ALE"
        int daily_export_count "🔒 ALE"
        date last_reset_date "🔒 ALE"
        string stripe_customer_id "🔒 ALE indexed"
        string stripe_subscription_id "🔒 ALE indexed"
        string subscription_status "🔒 ALE"
        datetime current_period_start "🔒 ALE"
        datetime current_period_end "🔒 ALE"
    }

    AUDIO_NOTES_GLOSSARY_RULES {
        uuid id PK
        string user_id FK "indexed"
        string misheard_term
        string correction
        bool is_active
        datetime deleted_at
        datetime created_at
    }

    AUDIO_NOTES_TRANSCRIPTION_LOGS {
        uuid id PK
        string user_id FK "indexed"
        bigint telegram_message_id "indexed"
        string file_hash "indexed"
        text original_text
        text summary
        datetime created_at
    }

    %% ============================================================
    %% Module: pdf_anki (KERA)
    %% ============================================================
    PDF_ANKI_USERS {
        string id PK "Google sub or anonymous"
        text email "indexed"
        text display_name
        text avatar_url
        text tier "free / premium / pro / payg / admin"
        text stripe_customer_id "indexed"
        text stripe_subscription_id "indexed"
        text subscription_status
        text balance_usd
        text payg
        text active_unified_pdf_model
        text monthly_pages_count
        text monthly_pages_window
    }

    PDF_ANKI_DOCUMENTS {
        string id PK
        string file_hash UK "indexed (SHA-256)"
        enum state "pending|processing|completed|completed_partial|failed"
        text error_message
        text resolved_model_id
        datetime created_at
        datetime updated_at
    }

    PDF_DECKS {
        uuid id PK
        string user_id FK "indexed"
        string document_id FK "indexed"
        string file_hash "indexed"
        text title_encrypted "🔒 ALE"
        enum status "completed|completed_partial"
        datetime created_at
        datetime updated_at
    }

    PDF_FLASHCARDS {
        uuid id PK
        uuid deck_id FK "indexed"
        text front_encrypted "🔒 ALE"
        text back_encrypted "🔒 ALE"
        int position
        datetime created_at
    }

    %% ============================================================
    %% Module: receipt_parser (VERA)
    %% ============================================================
    RECEIPT_PARSER_USERS {
        uuid id PK
        string user_id UK "indexed"
        text email
        text display_name
        text auth_provider
        text tier "free / premium / pro / payg"
        text balance_usd
        text stripe_customer_id "indexed"
        text stripe_subscription_id "indexed"
        text subscription_status
        text monthly_receipts_count
        text monthly_window_start
        text payg
        text active_receipt_parser_llm_model
    }

    RECEIPTS {
        uuid id PK
        uuid tenant_id "indexed (multi-tenancy)"
        uuid user_id FK "indexed"
        text user_role
        string user_role_fp "HMAC fingerprint indexed"
        text issuer_name
        string issuer_name_fp "HMAC fingerprint indexed"
        text payer_name
        vector issuer_embedding "1536 dim (pgvector)"
        string image_hash "indexed (SHA-256)"
        text invoice_number "indexed"
        text billing_id "indexed"
        text order_number "indexed"
        text nif_cif_ssn "indexed"
        text invoice_date "indexed"
        text due_date "indexed"
        text type "indexed"
        text currency
        text subtotal
        text discount
        text tax
        text shipping
        text total
        text items
        text taxes
        text payment_received
        text change_due
        text source
        text version
        text payload_encrypted "🔒 ALE (canonical receipt JSON)"
        datetime created_at
        datetime updated_at
    }

    %% ============================================================
    %% Relationships
    %% ============================================================
    AUDIO_NOTES_USERS ||--o{ AUDIO_NOTES_GLOSSARY_RULES : "owns"
    AUDIO_NOTES_USERS ||--o{ AUDIO_NOTES_TRANSCRIPTION_LOGS : "owns"
    PDF_ANKI_USERS ||--o{ PDF_DECKS : "owns"
    PDF_ANKI_DOCUMENTS ||--o{ PDF_DECKS : "produces"
    PDF_DECKS ||--o{ PDF_FLASHCARDS : "contains"
    RECEIPT_PARSER_USERS ||--o{ RECEIPTS : "owns"
    AUDIO_NOTES_USERS ||--o{ WALLET_TRANSACTIONS : "PAYG ledger"
    PDF_ANKI_USERS ||--o{ WALLET_TRANSACTIONS : "PAYG ledger"
    RECEIPT_PARSER_USERS ||--o{ WALLET_TRANSACTIONS : "PAYG ledger"
    FEEDBACK_LOG }|..|{ PROMPT_TEMPLATES : "linked by usage"
```

## Companion Stores (Not in this ERD)

The following stores are **not** SQLAlchemy entities and therefore do not appear in the diagram, but are part of the overall persistence story:

| Store | Backend | Purpose |
|---|---|---|
| `pdf_anki_chunks` | LanceDB (S3 / Cloudflare R2) | Per-chunk embeddings for L2 semantic deduplication. See [ADR-0009](../../adr/0009-postgres-pgvector-vs-pinecone.md). |
| `audio_processed:{hash}` | Redis | Content-addressable idempotency cache for audio. See [ADR-0013](../../adr/0013-sha256-idempotency-guard.md). |
| `pdf:deck:{file_hash}:{sig}` | Redis | Content-addressable cache for fully generated PDF decks. |
| Arq queue (`arq:queue:*`) | Redis | Background job queue (`process_pdf_task`, prune jobs, telegram delivery). |
| Active support tickets (`ticket:active:{ticket_id}`) | Redis | In-flight Telegram support conversation context. |
| Support attachments | S3-compatible object storage (or local disk in dev) | Files referenced by support tickets. |

## Key Conventions

* **Multi-tenancy:** Every read/write through repositories enforces `WHERE tenant_id = :tenant_id`. The only deliberate exception is the **content-addressable idempotency caches** (`audio_processed:{hash}`, `pdf:deck:{file_hash}:{sig}`) — see [CORE_INVARIANTS §2.2](../../architecture/CORE_INVARIANTS.md) and [ADR-0013](../../adr/0013-sha256-idempotency-guard.md).
* **HMAC fingerprints (`*_fp`):** Sensitive fields stored as ALE ciphertext are accompanied by deterministic HMAC-SHA256 fingerprints to keep them queryable without leaking plaintext.
* **Vector indices:** `feedback_log.embedding` and `receipts.issuer_embedding` use `pgvector` with HNSW. See [ADR-0002](../../adr/0002-use-pgvector.md).
* **Partitioning:** `feedback_log` is partitioned by month (`RANGE (created_at)`) to maintain sub-millisecond retrieval.
