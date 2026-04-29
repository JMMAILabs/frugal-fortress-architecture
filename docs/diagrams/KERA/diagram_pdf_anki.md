# PDF to Anki — Async RAG Flow

This sequence diagram illustrates the asynchronous processing of PDFs. It highlights the dual-pipeline architecture: Free/Admin tiers use a chunked RAG approach (LlamaParse + Groq), while Paid tiers use a direct multimodal approach (Vertex AI) for enhanced privacy and cohesion.
 
```mermaid
%%{init: {'theme': 'default', 'themeVariables': { 'noteBkgColor': '#fff5ad', 'noteTextColor': '#333333'}}}%%
sequenceDiagram
    participant User as Client (Frontend)
    participant API as FastAPI (Main Loop)
    participant Worker as Background Worker (Arq)
    participant Cache as Redis (Semantic Cache)
    participant Parser as LlamaParse API
    participant VectorDB as LanceDB (S3/R2)
    participant LLM as LiteLLM (Groq / Vertex)
    participant DB as Postgres (Supabase)

    User->>API: POST /api/v1/pdf/upload (File: biology.pdf, Tier)
    activate API
    API->>API: Generate File Hash
    API->>Worker: Enqueue Task (process_pdf, hash, tier)
    API-->>User: 202 Accepted { task_id: "12345", status: "processing" }
    deactivate API

    Note over User, API: User starts polling GET /status/12345

    activate Worker
    Worker->>Cache: Check Exact Hash Cache (L1)
    
    alt Cache Hit (Already Processed)
        Cache-->>Worker: Return Cached Flashcards Deck
        Worker->>DB: Update Document State (COMPLETED)
    else Cache Miss (New File)
        
        alt Tier: Free / Admin (Chunked Pipeline)
            Worker->>Parser: Upload PDF for Markdown Extraction
            Parser-->>Worker: Return Structured Markdown
            Worker->>Worker: Semantic Chunking
            
            loop For each Chunk
                Worker->>VectorDB: Vector Search (Semantic Deduplication)
                VectorDB-->>Worker: Similarity Score
                
                alt Concept is New
                    Worker->>LLM: Generate Flashcards (Groq Llama-3)
                    LLM-->>Worker: JSON: Front, Back, Tags
                    Worker->>VectorDB: Save Vectors
                else Concept Exists
                    Note right of Worker: Skip LLM generation (FinOps)
                end
            end
            
        else Tier: Premium / Pro / PAYG (Unified Multimodal Pipeline)
            Worker->>LLM: Send PDF Bytes + Prompt (Vertex AI Gemini)
            Note right of Worker: Zero Data Retention guaranteed by Vertex
            LLM-->>Worker: JSON: Full Deck (Front, Back, Tags)
        end
        
        Worker->>DB: Save Encrypted Deck (ALE)
        Worker->>Cache: Save Final Deck to Redis (TTL: 7 days)
    end
    
    Worker->>Worker: Mark Task "Completed"
    deactivate Worker

    User->>API: GET /api/v1/pdf/status/12345
    API-->>User: 200 OK { status: "completed", deck: [...] }
```
