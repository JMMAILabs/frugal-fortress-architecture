# C4 Model: Audio Notes Module

This document provides the C4 Model diagrams (Context, Container, and Component) for the Audio Notes module, highlighting the Idempotency Guard and the Nested Learning Loop.

## 1. System Context (Level 1)

```mermaid
C4Context
    title System Context: Audio Notes (Frugal Fortress)

    Person(user, "Telegram User", "Sends voice notes and text corrections via Telegram.")
    System(audio_notes, "Audio Notes System", "Transcribes, summarizes, and learns from user corrections.")
     
    System_Ext(telegram, "Telegram API", "Receives webhooks and sends markdown messages.")
    System_Ext(groq, "Groq API", "Fast Whisper transcription and Llama-3 summarization (Free Tier).")
    System_Ext(vertex_ai, "Google Vertex AI", "Multimodal audio processing (Premium/Pro/PAYG).")

    Rel(user, telegram, "Sends Audio/Text", "Mobile/Desktop App")
    Rel(telegram, audio_notes, "POST /api/v1/telegram/webhook", "HTTPS")
    Rel(audio_notes, telegram, "Sends Summary/Feedback", "HTTPS/REST")
    Rel(audio_notes, groq, "Transcribes & Summarizes", "HTTPS/REST")
    Rel(audio_notes, vertex_ai, "Direct Multimodal Inference", "HTTPS/gRPC")
```

## 2. Container Diagram (Level 2)
```mermaid

C4Container
    title Container Diagram: Audio Notes

    System_Ext(telegram, "Telegram API", "Webhook Provider")
    
    Container_Boundary(vpc, "Frugal Fortress VPC") {
        Container(api, "FastAPI Gateway", "Python 3.12", "Handles Telegram webhooks, auth, and routing.")
        
        ContainerDb(redis, "Redis", "In-Memory Cache", "Idempotency Guard (SHA-256), Rate Limiting, and Glossary Cache.")
        ContainerDb(postgres, "PostgreSQL", "Relational DB", "Stores User Profiles, Transcription Logs, and Glossary Rules.")
    }

    Rel(telegram, api, "Webhook Payload", "JSON/HTTPS")
    Rel(api, redis, "Check Audio Hash (Idempotency)", "RESP")
    Rel(api, postgres, "Fetch Glossary & Save Logs", "SQL/Asyncpg")

```

## 3. Component Diagram (Level 3 - Hexagonal Architecture)

```mermaid

C4Component
    title Component Diagram: Audio Notes (Hexagonal Architecture)

    Container_Boundary(module, "Module: audio_notes") {
        
        Component(router, "Telegram Webhook Router", "Primary Adapter", "Parses Telegram updates and routes to use cases.")
        
        Component(process_audio, "Process Audio Use Case", "Application Layer", "Orchestrates idempotency, transcription, and summarization.")
        Component(process_feedback, "Process Feedback Use Case", "Application Layer", "Evaluates user replies using the LLM Judge.")
        
        Component(ports, "Ports (Interfaces)", "Domain Layer", "IdempotencyPort, WhisperPort, SummarizerPort, CorrectionJudgePort.")
        
        Component(idempotency_adapter, "Redis Idempotency Guard", "Secondary Adapter", "Implements IdempotencyPort.")
        Component(whisper_adapter, "Whisper Groq Adapter", "Secondary Adapter", "Implements WhisperPort.")
        Component(vertex_adapter, "Vertex Audio Adapter", "Secondary Adapter", "Implements AudioToSummaryPort.")
        Component(judge_adapter, "LLM Correction Judge", "Secondary Adapter", "Implements CorrectionJudgePort.")
    }

    Rel(router, process_audio, "Invokes (New Audio)")
    Rel(router, process_feedback, "Invokes (Text Reply)")
    
    Rel(process_audio, ports, "Depends on")
    Rel(process_feedback, ports, "Depends on")
    
    Rel(idempotency_adapter, ports, "Implements")
    Rel(whisper_adapter, ports, "Implements")
    Rel(vertex_adapter, ports, "Implements")
    Rel(judge_adapter, ports, "Implements")
```