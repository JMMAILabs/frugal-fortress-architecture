# Hexagonal Module — Ports & Adapters

Internal structure of a module: Primary Adapters (REST, Worker), Application Layer, Domain, Secondary Ports, and Secondary Adapters.

```mermaid
%%{init: {'theme':'default', 'flowchart':{'nodeSpacing':50, 'rankSpacing':75}}}%%
graph TD
    subgraph Primary_Adapters ["Primary Adapters (Driving)"]
        Router["FastAPI Router<br/>HTTP Endpoints"]
        Worker["Arq Worker<br/>Background Tasks"]
    end

    subgraph Application_Layer["Application Layer (Use Cases)"]
        PortIn["Inbound Ports<br/>Interfaces"]
        Service["Application Service<br/>Orchestrator"]
    end

    subgraph Domain_Layer ["Domain Layer (Pure Python)"]
        Entities["Entities & Pydantic Models<br/>Flashcard, Receipt"]
        Logic["Business Logic<br/>MathValidator, ChunkingRules"]
    end

    subgraph Secondary_Ports ["Secondary Ports (Driven Interfaces)"]
        RepoPort["Repository Interface"]
        LLMPort["LLM Provider Interface"]
        CachePort["Cache Interface"]
    end

    subgraph Secondary_Adapters ["Secondary Adapters (Driven Implementations)"]
        Supabase["Supabase Adapter<br/>SQLAlchemy / pgvector"]
        LiteLLM["LiteLLM Adapter<br/>Groq / Vertex AI"]
        Redis["Redis Adapter<br/>Semantic Cache"]
    end

    %% Flow of Control
    Router --> PortIn
    Worker --> PortIn
    PortIn --> Service
    Service --> Entities
    Service --> Logic

    Service --> RepoPort
    Service --> LLMPort
    Service --> CachePort

    %% Dependency Inversion (Adapters implement Ports)
    Supabase -.->|Implements| RepoPort
    LiteLLM -.->|Implements| LLMPort
    Redis -.->|Implements| CachePort

    %% Subgraph styling
    style Domain_Layer fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px,color:#0d2818
    style Application_Layer fill:#fff3e0,stroke:#ef6c00,stroke-width:2px,color:#3e2723
    style Primary_Adapters fill:#e3f2fd,stroke:#1565c0,stroke-width:2px,color:#0a1f4b
    style Secondary_Adapters fill:#e3f2fd,stroke:#1565c0,stroke-width:2px,color:#0a1f4b
    style Secondary_Ports fill:#f3e5f5,stroke:#6a1b9a,stroke-width:2px,color:#2a063e,stroke-dasharray: 5 5

    %% Node styling — explicit dark text for dark-mode contrast
    classDef primary fill:#bbdefb,stroke:#0d47a1,stroke-width:1px,color:#0a1f4b
    classDef app fill:#ffe0b2,stroke:#e65100,stroke-width:1px,color:#3e2723
    classDef domain fill:#c8e6c9,stroke:#1b5e20,stroke-width:1px,color:#0d2818
    classDef port fill:#e1bee7,stroke:#4a148c,stroke-width:1px,color:#2a063e
    classDef adapter fill:#bbdefb,stroke:#0d47a1,stroke-width:1px,color:#0a1f4b
    class Router,Worker primary
    class PortIn,Service app
    class Entities,Logic domain
    class RepoPort,LLMPort,CachePort port
    class Supabase,LiteLLM,Redis adapter
```
