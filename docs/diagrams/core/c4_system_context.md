# System Context Diagram (C4 Level 1)

This diagram illustrates the **DefaultApp** within the enterprise ecosystem. It highlights the boundaries between the internal secure core and external SaaS providers.

```mermaid
flowchart TD
    %% Actors
    User["👤 Enterprise User"]
    Admin["👨‍💻 System Admin"]

    %% Core System
    subgraph "Enterprise Boundary (VPC)"
        API["🚀 DefaultApp API<br/>(FastAPI Core)"]
        Worker["⚙️ Async Worker<br/>(Arq/Redis)"]
        DB[("🗄️ PostgreSQL<br/>pgvector")]
        Cache[("⚡ Redis<br/>Cache & Queue")]
    end

    %% External Systems
    subgraph "External SaaS Providers"
        OpenAI["🤖 OpenAI / Groq API<br/>(LLM Inference)"]
        Auth0["🛡️ Identity Provider<br/>(OAuth2 / JWT)"]
        Grafana["📊 Observability Cloud<br/>(Logs/Metrics)"]
    end

    %% Relationships
    User -->|"HTTPS/REST"| API
    Admin -->|"HTTPS/GraphQL"| API

    API -->|"Auth Check"| Auth0
    API -->|"Read/Write"| DB
    API -->|"Cache/PubSub"| Cache

    API -->|"Inference (Circuit Breaker)"| OpenAI

    Worker -->|"Process Jobs"| Cache
    Worker -->|"Persist Logs"| DB

    API -.->|"OTLP Traces"| Grafana
    Worker -.->|"OTLP Traces"| Grafana

    %% Styling
    style API fill:#e1f5fe,stroke:#01579b,stroke-width:2px
    style Worker fill:#fff9c4,stroke:#fbc02d,stroke-width:2px
    style OpenAI fill:#f3e5f5,stroke:#7b1fa2,stroke-dasharray: 5 5
```
