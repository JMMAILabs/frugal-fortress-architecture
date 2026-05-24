# Backend Monolith — System Context

High-level view of the Backend Monolith (Railway / FastAPI), Core, Shared Utilities, and Modules with API routes and external services. The diagram only references infrastructure that actually exists in `src/`.

Split into three focused diagrams so each is legible on GitHub:

1. **Gateway + Core + Shared Utilities** — the "spine" of the monolith.
2. **Modules + External Providers** — which module talks to which SaaS.
3. **API route map** — `/api/v1/...` → module routing.

---

## 1. Gateway + Core + Shared Utilities

```mermaid
%%{init: {'theme':'default', 'flowchart':{'nodeSpacing':50, 'rankSpacing':70}}}%%
flowchart TD
    subgraph Frontend["Frontend (Vercel)"]
        FE[Next.js Apps<br/>genui · pdf-anki · receipts]
    end

    subgraph Backend ["Backend Monolith — Railway · FastAPI"]
        API_GATEWAY([FastAPI App])

        subgraph Core ["Core"]
            C_CONFIG[Configuration<br/>pydantic-settings]
            C_DB[(Postgres / Supabase<br/>+ pgvector)]
            C_REDIS[(Redis<br/>cache · queue · rate)]
            C_LANCE[(LanceDB<br/>S3 / Cloudflare R2)]
        end

        subgraph Shared ["Shared Utilities"]
            S_LLM[LiteLLM Client]
            S_AUTH[JWT / OAuth Middleware]
            S_FINOPS[FinOps Callback]
            S_OTEL[OpenTelemetry / structlog]
        end
    end

    FE --> API_GATEWAY
    API_GATEWAY --- C_CONFIG
    API_GATEWAY --- C_DB
    API_GATEWAY --- C_REDIS
    API_GATEWAY --- C_LANCE
    API_GATEWAY --- S_LLM
    API_GATEWAY --- S_AUTH
    API_GATEWAY --- S_FINOPS
    API_GATEWAY --- S_OTEL

    classDef gw fill:#e1f5fe,stroke:#01579b,stroke-width:2px,color:#01323b
    classDef core fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px,color:#0d2818
    classDef shared fill:#fff3e0,stroke:#ef6c00,stroke-width:2px,color:#3e2723
    classDef fe fill:#f3e5f5,stroke:#6a1b9a,stroke-width:2px,color:#2a063e
    class API_GATEWAY gw
    class C_CONFIG,C_DB,C_REDIS,C_LANCE core
    class S_LLM,S_AUTH,S_FINOPS,S_OTEL shared
    class FE fe
```

---

## 2. Modules and External Providers

```mermaid
%%{init: {'theme':'default', 'flowchart':{'nodeSpacing':50, 'rankSpacing':80}}}%%
flowchart LR
    subgraph Modules ["Modules (src/app/modules/)"]
        direction TB
        M_AUDIO["audio_notes / AURA"]
        M_PDF["pdf_anki / KERA"]
        M_RECEIPTS["receipt_parser / VERA"]
        M_GENUI["genui"]
        M_DLP["dlp_proxy<br/>local SLM PII guard"]
    end

    subgraph Providers ["External Providers"]
        direction TB
        EXT_GROQ["Groq API<br/>Whisper + Llama-3.3"]
        EXT_VERTEX["Google Vertex AI<br/>Gemini 2.5"]
        EXT_LLAMAPARSE["LlamaParse Cloud<br/>PDF / image OCR"]
        EXT_TG["Telegram Bot API"]
        EXT_GOOGLE["Google OAuth2 / OIDC"]
        EXT_STRIPE["Stripe billing"]
    end

    M_AUDIO -->|Free: Whisper + Llama-3.3| EXT_GROQ
    M_AUDIO -->|Paid: Gemini multimodal| EXT_VERTEX
    M_AUDIO -->|Delivery| EXT_TG

    M_PDF -->|Free: PDF to markdown| EXT_LLAMAPARSE
    M_PDF -->|Free: flashcard gen| EXT_GROQ
    M_PDF -->|Paid: unified multimodal| EXT_VERTEX

    M_RECEIPTS -->|Free: OCR| EXT_LLAMAPARSE
    M_RECEIPTS -->|Free: JSON extraction| EXT_GROQ
    M_RECEIPTS -->|Paid: multimodal| EXT_VERTEX

    M_GENUI -->|Proposal LLM| EXT_GROQ

    %% Auth & billing (gateway-wide, but shown for completeness)
    M_RECEIPTS -.->|OAuth2 / billing| EXT_GOOGLE
    M_RECEIPTS -.->|Stripe webhooks| EXT_STRIPE
    M_PDF -.->|OAuth2 / billing| EXT_GOOGLE
    M_PDF -.->|Stripe webhooks| EXT_STRIPE
    M_AUDIO -.->|Stripe webhooks| EXT_STRIPE

    classDef module fill:#e1f5fe,stroke:#01579b,stroke-width:2px,color:#01323b
    classDef llm fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px,color:#3a0a4b,stroke-dasharray: 5 5
    classDef saas fill:#fff3e0,stroke:#ef6c00,stroke-width:2px,color:#3e2723,stroke-dasharray: 5 5
    class M_AUDIO,M_PDF,M_RECEIPTS,M_GENUI,M_DLP module
    class EXT_GROQ,EXT_VERTEX,EXT_LLAMAPARSE llm
    class EXT_TG,EXT_GOOGLE,EXT_STRIPE saas
```

---

## 3. API Route Map

```mermaid
%%{init: {'theme':'default', 'flowchart':{'nodeSpacing':40, 'rankSpacing':60}}}%%
flowchart LR
    API([FastAPI App])

    API -->|/api/v1/audio<br/>/api/v1/telegram/webhook| M_AUDIO[audio_notes]
    API -->|/api/v1/pdf| M_PDF[pdf_anki]
    API -->|/api/v1/receipts| M_RECEIPTS[receipt_parser]
    API -->|/api/v1/genui| M_GENUI[genui]
    API -->|/api/v1/dlp| M_DLP[dlp_proxy]

    %% Per-module stores
    M_AUDIO -.->|SHA-256 idempotency| R1[(Redis)]
    M_AUDIO -.->|Glossary · transcripts| D1[(Postgres)]
    M_PDF -.->|L1 deck cache · queue| R2[(Redis)]
    M_PDF -.->|Decks · users · feedback| D2[(Postgres)]
    M_PDF -.->|L2 semantic dedup chunks| L1[(LanceDB)]
    M_RECEIPTS -.->|Encrypted receipts ALE| D3[(Postgres)]
    M_GENUI -.->|Static metadata| D4[(Postgres)]

    classDef gw fill:#e1f5fe,stroke:#01579b,stroke-width:2px,color:#01323b
    classDef module fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px,color:#0d2818
    classDef store fill:#e0f2f1,stroke:#00695c,stroke-width:1px,color:#0a3531
    class API gw
    class M_AUDIO,M_PDF,M_RECEIPTS,M_GENUI,M_DLP module
    class R1,R2,D1,D2,D3,D4,L1 store
```

> **Provider matrix.** Free tier: Groq (`whisper-large-v3`, `llama-3.3-70b-versatile`) + LlamaParse (PDFs / images). Paid tiers (Premium / Pro / PAYG): Google Vertex AI (`gemini-2.5-flash-lite`, `gemini-2.5-flash`, `gemini-2.5-pro`). **No OpenAI, Anthropic, or Cohere models or APIs are used**; all reranking is local ONNX (`reranker_adapter.py`). All outbound LLM calls flow through a Redis-backed circuit breaker; see [Circuit Breaker State Machine](circuit_breaker_state.md).
