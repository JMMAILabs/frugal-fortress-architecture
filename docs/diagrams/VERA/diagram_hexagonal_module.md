# Receipt Parser — Hexagonal Architecture & Flows

This document details the internal architecture of the Receipt Parser module, mapping the strict boundaries between the Domain, Application, and Infrastructure layers, followed by the sequence diagrams for its core operations.

## 1. Hexagonal Module Wiring

```mermaid
flowchart LR
    %% -----------------------------
    %% Receipt Parser (hexagonal)
    %% -----------------------------

    %% Ports
    subgraph P[Ports Domain/Application]
        DS["Domain Schemas + Entities"]
        MV["Math Validator (dates + totals)"]
        SCL["Self-Correction Loop (206/OK)"]
        APP["Receipt Processor (tier routing)"]
    end

    %% Adapters Inbound
    subgraph IN[Inbound Adapters]
        RT["FastAPI Router /receipts/process"]
        AUTH["Auth Adapter /auth/login/google + callback"]
    end

    %% Adapters Outbound
    subgraph OUT[Outbound Adapters]
        REPO[("ReceiptRepository (ALE encrypted/decrypted)")]
        USERREPO[("ReceiptParserUserRepository")]
        LPARSE["LlamaParseAdapter (free tier)"]
        VLMVLM["VLM / LLM (LiteLLMProvider)"]
        EMB["Embedding (llm.embed / FastEmbed local)"]
        TOK["TokenManagerPort (count_tokens)"]
        COST["CostCalculatorPort"]
    end

    %% Shared services
    subgraph SEC[Security / Privacy]
        TOKR["PII Tokenizer Service (Presidio + regex)"]
    end

    %% Wiring / Direction
    RT -->|Receives image_base64, user_role, image_hash| APP
    AUTH -->|Generates JWT + user_id| RT

    APP --> DS
    APP --> MV
    APP --> SCL

    %% LLM routing
    APP -->|Free tier| LPARSE
    APP -->|Paid tiers| VLMVLM
    APP --> TOK
    APP --> COST

    %% Few-shot via repo
    APP -->|get_by_issuer_name| REPO
    REPO -->|SQLite: exact issuer_name| REPO
    REPO -->|Postgres: cosine_distance on issuer_embedding| REPO

    %% Tokenization before LLM (free tier)
    APP -->|free| TOKR
    TOKR --> VLMVLM

    %% Persistence
    APP -->|save verified receipt| REPO
    REPO --> USERREPO

    %% Embedding computation happens via llm.embed in router
    RT -->|llm.embed to issuer_embedding| REPO
```


## 2. Processing Sequence (Self-Correction Loop)
```mermaid
sequenceDiagram
    autonumber
    actor F as Frontend (Edge/PWA)
    participant API as FastAPI `/receipts/process`
    participant PROC as process_receipt_image
    participant LPARSE as LlamaParseAdapter
    participant PIITOK as PIITokenizerService (Presidio/regex)
    participant LLM as LLMProvider (Groq / VLM)
    participant MATH as validate_receipt_math
    participant SCLOOP as SelfCorrectionLoop (Max 3)
    participant REPO as ReceiptRepository (ALE Encrypted)

    F->>API: POST /api/v1/receipts/process\nimage_base64, user_role, image_hash\nHeader: X-User-Id
    API->>API: Load tier (ReceiptParserUserRepository)\nEnforce limits (429)
    API->>API: Check duplicates (409) via image_hash
    API->>PROC: process_receipt_image(tier, image_base64, user_stated_role)

    alt Tier uses Free Stack (Free/Admin)
        PROC->>LPARSE: extract_markdown(image_bytes)
        PROC->>PIITOK: tokenize(OCR markdown)
        PROC->>SCLOOP: self-correction loop (MAX_RETRIES=3)\nLLM JSON + validate schema + validate math
        SCLOOP->>LLM: LlamaParse text -> Groq JSON
        SCLOOP->>MATH: validate_receipt_math(receipt)
    else Paid Tier (Premium/Pro/PAYG)
        PROC->>SCLOOP: self-correction loop (MAX_RETRIES=3)\nLLM JSON + validate schema + validate math
        SCLOOP->>LLM: Direct VLM (Image Data URL) -> JSON
        SCLOOP->>MATH: validate_receipt_math(receipt)
    end

    alt Math OK (Before retries exhausted)
        SCLOOP-->>PROC: Verified Receipt
        PROC->>REPO: save(receipt) (encrypt payload + plaintext issuer_name)\n(issuer_embedding optional)
        REPO-->>PROC: receipt_id
        PROC-->>API: ReceiptProcessResponse {status: "verified"}
        API-->>F: HTTP 200 + `receipt`
    else Math Fails (After 3 retries)
        SCLOOP-->>PROC: best_effort_receipt (Last JSON attempt)
        PROC-->>API: ReceiptProcessResponse {status: requires_human_correction}
        API-->>F: HTTP 206 + `best_effort_receipt` (Editable)
    end
```

## 3. Few-Shot Vendor Matching (RAG)
```mermaid
sequenceDiagram
    autonumber
    actor API as Router `/receipts/process`
    participant RT as process_receipt_image
    participant REPO as ReceiptRepository.get_by_issuer_name
    participant EMB as llm.embed(issuer_name)
    participant BUILD as _build_system_with_examples
    participant LLM as LLMProvider

    API->>EMB: llm.embed(issuer_name) -> issuer_embedding (router)
    API->>REPO: get_by_issuer_name(user_id, issuer_name, limit=2, issuer_embedding)

    alt Postgres Available
        REPO->>REPO: cosine_distance(issuer_embedding)\nORDER BY similarity + fill exact match
    else SQLite (Test Environment)
        REPO->>REPO: Exact match on issuer_name
    end

    REPO-->>API: Decrypted receipts list (max 2)
    API->>RT: process_receipt_image(..., get_examples=_get_examples)
    RT->>BUILD: Build system prompt + `<example>` blocks\n(Injects up to 2 receipts)
    RT->>LLM: Generate JSON consistent with provider format
```

## 4. Authentication Flow (Google OAuth)
```mermaid
sequenceDiagram
    autonumber
    actor F as Frontend
    participant AUTH as FastAPI `/auth/login/google` & `/auth/callback/google`
    participant G as Google OAuth
    participant DB as Supabase/Postgres `receipt_parser_users`

    F->>AUTH: GET /api/v1/auth/login/google
    AUTH-->>G: Redirect to consent screen (302) with redirect_uri + state
    G-->>AUTH: GET /api/v1/auth/callback/google?code=...&state=...
    AUTH->>AUTH: Exchange code -> access_token (HTTP POST)
    AUTH->>AUTH: GET userinfo (openid email profile)
    AUTH->>DB: upsert_google_oauth_user\nuser_id = google sub\nemail/display_name/auth_provider
    DB-->>AUTH: row (tier, balance_usd, ...)
    AUTH-->>F: JWT Bearer + user_id (sub)
    F->>F: Persist JWT + user_id
    F->>AUTH: GET /api/v1/auth/me (Optional)
    F->>AUTH: Use X-User-Id=user_id in POST /receipts/process
```