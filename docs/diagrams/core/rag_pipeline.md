# RAG & Hybrid Search Pipeline

This diagram details the **Retrieval-Augmented Generation (RAG)** logic executed by the `AdaptiveRouter`. It showcases the "Hybrid Search" strategy (Vector + Keyword) and the "Reranking" step for maximum precision.

```mermaid
%%{init: {'theme':'default', 'flowchart':{'nodeSpacing':45, 'rankSpacing':65}}}%%
flowchart TD
    Start([User Query]) --> PII["🛡️ PII Scrubber"]
    PII --> Guard["🔒 Prompt Guard"]

    Guard -->|Safe| Router{"Adaptive Router"}
    Guard -->|Unsafe| Block(["🚫 Block Request"])

    subgraph Retrieval ["Retrieval Layer (Parallel)"]
        Router -->|1. Generate Embedding| Embed["FastEmbed<br/>local ONNX"]

        Embed -->|Vector Search| PG_Vec[("Postgres<br/>pgvector")]
        Router -->|Keyword Search| PG_Text[("Postgres<br/>TSVECTOR")]
    end

    PG_Vec -->|Top-K Semantic| Merger
    PG_Text -->|Top-K Exact| Merger

    subgraph Refinement ["Refinement Layer"]
        Merger["🔄 RRF Fusion<br/>Normalize Scores"]
        Merger --> Rerank["🧠 Cross-Encoder Reranker<br/>local ONNX · no Cohere"]
        Rerank --> Context["📝 Context Window Builder<br/>Token Trimming"]
    end

    Context --> LLM["🤖 LLM Inference<br/>Vertex AI Gemini / Groq GPT-OSS 120B"]
    LLM --> Stream(["⚡ Stream Response"])

    %% Subgraph styling
    style Retrieval fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px,color:#0d2818
    style Refinement fill:#fff3e0,stroke:#ef6c00,stroke-width:2px,color:#3e2723

    %% Node styling — explicit dark text for dark-mode readability
    classDef start fill:#c8e6c9,stroke:#1b5e20,stroke-width:2px,color:#0d2818
    classDef guard fill:#ffcdd2,stroke:#b71c1c,stroke-width:1px,color:#3e0a0a
    classDef router fill:#ffe0b2,stroke:#e65100,stroke-width:2px,color:#3e2723
    classDef compute fill:#bbdefb,stroke:#0d47a1,stroke-width:1px,color:#0a1f4b
    classDef store fill:#b2dfdb,stroke:#004d40,stroke-width:1px,color:#0a3531
    classDef rerank fill:#e1bee7,stroke:#4a148c,stroke-width:2px,color:#2a063e
    classDef llm fill:#fff9c4,stroke:#f57f17,stroke-width:1px,color:#3e2723
    classDef block fill:#ef9a9a,stroke:#b71c1c,stroke-width:2px,color:#3e0a0a
    class Start,Stream start
    class PII,Guard guard
    class Router router
    class Embed,Merger,Context compute
    class PG_Vec,PG_Text store
    class Rerank rerank
    class LLM llm
    class Block block
```
