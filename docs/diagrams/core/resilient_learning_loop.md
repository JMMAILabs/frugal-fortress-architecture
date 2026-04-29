# Resilient Learning Loop (Sequence Diagram)

This diagram details the **Request Flow** within the system, highlighting the **Multi-Tier Caching** strategy, **Circuit Breakers**, and the **Async Persistence** mechanism (FinOps).

```mermaid
sequenceDiagram
    autonumber
    participant Client
    participant Middleware as 🛡️ Security & Audit
    participant Router as 🧠 Adaptive Router
    participant Cache as ⚡ Redis Cache
    participant VectorDB as 🗄️ Postgres (pgvector)
    participant LLM as 🤖 LLM Gateway
    participant Worker as 👷 Async Worker

    Client->>Middleware: POST /api/v1/stream
    Middleware->>Middleware: PII Scrubbing & Rate Limiting
    Middleware->>Router: Route Request

    rect rgb(255, 240, 240)
        Note over Router, Cache: 1. Security & Fast Path
        Router->>Router: PromptGuard (Injection Check)
        Router->>Cache: Check Exact Match (L1)
        alt Exact Match Found
            Cache-->>Router: Return Cached Response
            Router-->>Client: Stream Response (Immediate)
        end
    end

    rect rgb(240, 248, 255)
        Note over Router, VectorDB: 2. Semantic Retrieval (RAG)
        Router->>LLM: Generate Embedding
        Router->>VectorDB: Hybrid Search (Vector + Keyword)
        VectorDB-->>Router: Top-K Documents
        Router->>Router: Rerank Results (Cross-Encoder)
    end

    rect rgb(240, 255, 240)
        Note over Router, LLM: 3. Inference & Resilience
        Router->>Router: Construct System Prompt
        Router->>LLM: Stream Completion (Circuit Breaker Protected)
        loop Stream Chunks
            LLM-->>Router: Chunk + Usage Metadata
            Router-->>Client: SSE Chunk
        end
    end

    rect rgb(255, 250, 205)
        Note over Router, Worker: 4. Async Persistence (FinOps)
        Router->>Worker: Publish InteractionLog (Event Bus)
        Note right of Worker: Decoupled from User Latency
        Worker->>VectorDB: Persist Log & Cost
    end
```
