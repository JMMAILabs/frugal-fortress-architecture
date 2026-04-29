# PAYG Wallet — Dynamic Billing & Two-Phase Commit (2PC)

This sequence diagram shows the FinOps lifecycle for a Pay-As-You-Go user, highlighting the atomic Two-Phase Commit (ADR-0012) that prevents Denial of Wallet (DoW) attacks.
 
```mermaid
%%{init: {'theme': 'default', 'themeVariables': { 'noteBkgColor': '#e1f5fe', 'noteTextColor': '#01579b'}}}%%
sequenceDiagram
    autonumber
    participant User as Telegram User
    participant API as FastAPI
    participant Budget as BudgetManager (Redis Lua)
    participant LLM as Vertex AI (Gemini 2.5 Pro)
    participant DB as PostgreSQL (Wallet)

    Note over User, DB: Phase 1: Pre-flight Reservation (Reserve)
    User->>API: Send 15-minute Audio Note
    API->>API: Calculate Estimated Cost (e.g., $0.20)
    API->>Budget: reserve_budget(tenant_id, $0.20)
    
    alt Insufficient Funds
        Budget-->>API: Return -1 (Rejected)
        API-->>User: "Insufficient balance for Gemini 2.5 Pro. Falling back to base tier."
    else Funds Available
        Budget-->>API: Return 1 (Reserved)
        
        Note over API, LLM: Phase 2: Inference
        API->>LLM: Process Multimodal Audio
        LLM-->>API: JSON Summary + Exact Token Usage
        
        Note over API, DB: Phase 3: Exact Deduction (Commit)
        API->>API: Calculate Exact Cost + 40% Markup (e.g., $0.18)
        API->>DB: SELECT FOR UPDATE -> Deduct $0.18 -> Insert DEBIT Tx
        API->>Budget: commit_budget(tenant_id, reserved=$0.20, actual=$0.18)
        API-->>User: Send Summary + "Cost: $0.18 | Remaining: $9.82"
    end