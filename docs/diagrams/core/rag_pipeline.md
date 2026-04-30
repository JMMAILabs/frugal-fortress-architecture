# RAG & Hybrid Search Pipeline

This diagram details the **Retrieval-Augmented Generation (RAG)** logic executed by the `AdaptiveRouter`. It showcases the "Hybrid Search" strategy (Vector + Keyword) and the "Reranking" step for maximum precision.

```mermaid
flowchart TD
    Start([User Query]) --> PII["🛡️ PII Scrubber"]
    PII --> Guard["🔒 Prompt Guard"]

    Guard -->|Safe| Router{"Adaptive Router"}
    Guard -->|Unsafe| Block(["🚫 Block Request"])

    subgraph "Retrieval Layer (Parallel)"
        Router -->|1. Generate Embedding| Embed["FastEmbed (local ONNX)"]

        Embed -->|Vector Search| PG_Vec[("Postgres<br/>pgvector")]
        Router -->|Keyword Search| PG_Text[("Postgres<br/>TSVECTOR")]
    end

    PG_Vec -->|Top-K Semantic| Merger
    PG_Text -->|Top-K Exact| Merger

    subgraph "Refinement Layer"
        Merger["🔄 RRF Fusion<br/>Normalize Scores"]
        Merger --> Rerank["🧠 Cross-Encoder Reranker<br/>(local ONNX, no Cohere)"]
        Rerank --> Context["📝 Context Window Builder<br/>(Token Trimming)"]
    end

    Context --> LLM["🤖 LLM Inference<br/>(Vertex AI Gemini / Groq Llama-3.3)"]
    LLM --> Stream(["⚡ Stream Response"])

    %% Styling
    style Router fill:#ffe0b2,stroke:#e65100
    style Rerank fill:#e1bee7,stroke:#4a148c
    style PG_Vec fill:#b2dfdb,stroke:#004d40
    style PG_Text fill:#b2dfdb,stroke:#004d40
```
