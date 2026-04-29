# Receipt Parser — Edge Compute & Self-Correction Sequence

This sequence diagram illustrates the hybrid architecture: Edge computing for image compression, Cloud orchestration for AI extraction, deterministic Math Validation, and the Human-in-the-Loop (HITL) fallback using HTTP 206.

```mermaid
%%{init: {'theme': 'default', 'themeVariables': { 'noteBkgColor': '#fff5ad', 'noteTextColor': '#333333'}}}%%
sequenceDiagram
    autonumber
    participant Edge as Next.js PWA (Browser)
    participant API as FastAPI Backend
    participant PII as PII Scrubber (Presidio)
    participant LLM as LiteLLM (Groq / Vertex)
    participant Math as Math Validator (Python)
    participant DB as Postgres (pgvector)

    Note over Edge, API: Phase 1: Edge Compute & Upload
    Edge->>Edge: Capture Photo & Compress via Canvas API (5MB -> 150KB)
    Edge->>API: POST /receipts/process (Base64 Image)
    activate API
    
    API->>DB: Fetch Few-Shot Examples (pgvector similarity on Issuer)
    
    Note over API, Math: Phase 2: Self-Correction Loop (Max 3 Retries)
    loop Until Math is Valid or Max Retries Reached
        alt Free Tier
            API->>PII: Scrub PII from OCR Text
            API->>LLM: Extract JSON (Groq Llama-3)
        else Paid Tier
            API->>LLM: Extract JSON from Image (Vertex Gemini VLM)
        end
        
        LLM-->>API: Structured JSON Receipt
        API->>Math: validate_receipt_math(receipt)
        
        alt Math is Invalid
            Math-->>API: Error: "Subtotal + Tax != Total"
            API->>API: Append Error to Prompt for next retry
        end
    end

    Note over API, Edge: Phase 3: Response & Human-in-the-Loop
    alt Math is Valid
        API->>DB: Save Encrypted Receipt (ALE)
        API-->>Edge: HTTP 200 OK { status: "verified", receipt: {...} }
    else Math is Invalid (After 3 Retries)
        API-->>Edge: HTTP 206 Partial Content { status: "requires_human_correction", best_effort_receipt: {...} }
        Note left of Edge: UI renders form with AI's best effort data
        Edge->>Edge: User manually corrects the math error
        Edge->>API: POST /receipts/history (Corrected JSON)
        API->>Math: validate_receipt_math(corrected_receipt)
        API->>DB: Save Encrypted Receipt & Update Vector Store
        API-->>Edge: HTTP 200 OK { status: "saved" }
    end
    deactivate API