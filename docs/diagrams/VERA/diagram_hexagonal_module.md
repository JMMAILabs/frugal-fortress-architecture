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
  participant SCLOOP as SelfCorrectionLoop (hasta 3)
  participant REPO as ReceiptRepository (ALE cifrado)

  F->>API: POST /api/v1/receipts/process\nimage_base64, user_role, image_hash\nHeader: X-User-Id
  API->>API: Carga tier (ReceiptParserUserRepository)\nEnforce límites (429)
  API->>API: Duplicados (409) por image_hash
  API->>PROC: process_receipt_image(tier, image_base64, user_stated_role)
  
  alt Tier usa stack free (Free/Admin)
    PROC->>LPARSE: extract_markdown(image_bytes)
    PROC->>PIITOK: tokenize(OCR markdown)
    PROC->>SCLOOP: self-correction loop (MAX_RETRIES=3)\nLLM JSON + validate schema + validate math
    SCLOOP->>LLM: LlamaParse text -> Groq JSON
    SCLOOP->>MATH: validate_receipt_math(receipt)
  else Tier pago (Premium/Pro/Ultra)
    PROC->>SCLOOP: self-correction loop (MAX_RETRIES=3)\nLLM JSON + validate schema + validate math
    SCLOOP->>LLM: Direct VLM (imagen en data URL) -> JSON
    SCLOOP->>MATH: validate_receipt_math(receipt)
  end
  
  alt Math OK (antes de agotar reintentos)
    SCLOOP-->>PROC: receipt verificado
    PROC->>REPO: save(receipt) (encrypt payload + issuer_name claro)\n(issuer_embedding opcional)
    REPO-->>PROC: receipt_id
    PROC-->>API: ReceiptProcessResponse {status: "verified"}
    API-->>F: HTTP 200 + `receipt`
  else Math falla tras 3 reintentos
    SCLOOP-->>PROC: best_effort_receipt (JSON último)
    PROC-->>API: ReceiptProcessResponse {status: requires_human_correction}
    API-->>F: HTTP 206 + `best_effort_receipt` editable
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
  
  alt Postgres disponible
    REPO->>REPO: cosine_distance(issuer_embedding)\nORDER BY para similitud + fill exact match
  else SQLite (tests)
    REPO->>REPO: coincidencia exacta issuer_name
  end
  
  REPO-->>API: receipts list (max 2) ya descifradas
  API->>RT: process_receipt_image(..., get_examples=_get_examples)
  RT->>BUILD: build system prompt + `<example>` blocks\n(inyecta hasta 2 recibos)
  RT->>LLM: completa con JSON coherente con el proveedor
```

## 4. Authentication Flow (Google OAuth)
```mermaid
sequenceDiagram
  autonumber
  actor F as Frontend
  participant AUTH as FastAPI `/auth/login/google` y `/auth/callback/google`
  participant G as Google OAuth
  participant DB as Supabase/Postgres `receipt_parser_users`

  F->>AUTH: GET /api/v1/auth/login/google
  AUTH-->>G: Redirect a consent screen (302) con redirect_uri + state
  G-->>AUTH: GET /api/v1/auth/callback/google?code=...&state=...
  AUTH->>AUTH: Exchange code -> access_token (HTTP POST)
  AUTH->>AUTH: GET userinfo (openid email profile)
  AUTH->>DB: upsert_google_oauth_user\nuser_id = google sub\nemail/display_name/auth_provider
  DB-->>AUTH: row (tier, balance_usd, ...)
  AUTH-->>F: JWT Bearer + user_id (sub)
  F->>F: Persist JWT + user_id
  F->>AUTH: GET /api/v1/auth/me (opcional)
  F->>AUTH: usar X-User-Id=user_id en POST /receipts/process
```