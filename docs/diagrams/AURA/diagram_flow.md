# Audio Notes — Idempotency & Nested Learning Loop

This sequence diagram illustrates the dual-path processing (Free vs Paid) and the self-correcting feedback loop (Reply-to-Correct), highlighting the strict Hexagonal Architecture boundaries and Application-Level Encryption (ALE).

```mermaid
%%{init: {'theme': 'default', 'themeVariables': { 'noteBkgColor': '#e3f2fd', 'noteTextColor': '#0d47a1', 'actorBkg': '#eceff1'}}}%%
sequenceDiagram
    autonumber
    participant User as Telegram User
    participant API as FastAPI (Webhook)
    participant Cache as Redis (Idempotency)
    participant DB as Postgres (ALE Encrypted)
    participant LLM as LiteLLM (Groq/Vertex)

    Note over User, LLM: Phase 1: Audio Processing & Idempotency Guard
    User->>API: Sends Voice Note
    activate API
    API->>API: Calculate SHA-256 Hash of Audio Bytes
    API->>Cache: Check Hash (RedisIdempotencyGuard)
    
    alt Cache Hit (Viral/Duplicate Audio)
        Cache-->>API: Return Cached Summary
        API-->>User: Send Summary (Latency < 50ms, Cost $0)
    else Cache Miss (New Audio)
        API->>DB: Fetch User Glossary Rules (RAG)
        
        alt Tier: Free
            API->>LLM: Transcribe (WhisperGroqAdapter)
            LLM-->>API: Transcript
            API->>LLM: Summarize with Glossary (LLMSummarizerAdapter)
            LLM-->>API: JSON Summary
        else Tier: Premium/Pro/PAYG
            API->>LLM: Multimodal Audio + Glossary (VertexAudioToSummaryAdapter)
            Note right of API: Zero Data Retention Guaranteed
            LLM-->>API: JSON Summary + Transcript
        end
        
        API->>Cache: Save to Cache (TTL based on Tier)
        API->>DB: Save Transcription Log (AES-128-CBC Encrypted)
        API-->>User: Send Markdown Summary
    end
    deactivate API

    Note over User, LLM: Phase 2: Nested Learning Loop (Self-Correction)
    User->>API: Reply to Summary: "It's FastAPI, not Fast Happy"
    activate API
    API->>DB: Fetch Original Transcript Log (Decrypt ALE)
    API->>LLM: LLMCorrectionJudgeAdapter: Evaluate Correction vs Original
    LLM-->>API: {"valid": true, "misheard": "Fast Happy", "correction": "FastAPI"}
    API->>DB: Save to Glossary Rules (AES-128-CBC Encrypted)
    API->>Cache: Invalidate User Glossary Cache
    API-->>User: "Correction learned! I will remember this."
    deactivate API
```