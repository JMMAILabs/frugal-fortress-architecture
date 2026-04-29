# C4 Model: PDF to Anki Module

This document provides the C4 Model diagrams (Context, Container, and Component) for the PDF to Anki AI Copilot module, highlighting the strict Hexagonal Architecture boundaries.

## 1. System Context (Level 1)
Shows how the user interacts with the system and the external dependencies.

```mermaid
C4Context
    title System Context: PDF to Anki AI Copilot (Frugal Fortress)

    Person(user, "Student / Professional", "Uploads PDFs to generate study flashcards.")
    System(pdf_anki, "PDF to Anki System", "Extracts text, chunks semantically, and generates Anki flashcards using AI.")
    
    System_Ext(llamaparse, "LlamaParse API", "Extracts high-quality markdown from complex PDFs (Free tiers).")
    System_Ext(vertex_ai, "Google Vertex AI", "Multimodal LLM processing (Premium/Pro/PAYG tiers) with Zero Data Retention.")
    System_Ext(groq, "Groq API", "Fast LLM inference for free-tier flashcard generation.")

    Rel(user, pdf_anki, "Uploads PDF & Reviews Flashcards", "HTTPS")
    Rel(pdf_anki, llamaparse, "Sends PDF bytes, gets Markdown", "HTTPS/REST")
    Rel(pdf_anki, vertex_ai, "Sends PDF/Text, gets JSON Flashcards", "HTTPS/gRPC")
    Rel(pdf_anki, groq, "Sends Text, gets JSON Flashcards", "HTTPS/REST")

```

## 2. Container Diagram (Level 2)

Shows the high-level technical containers within the Frugal Fortress monolith.

```mermaid
C4Container
    title Container Diagram: PDF to Anki AI Copilot

    Person(user, "User", "Uploads PDF")
    
    Container_Boundary(vpc, "Frugal Fortress VPC") {
        Container(frontend, "Next.js Frontend", "React, Tailwind", "Provides the UI, handles edge-validation and polling.")
        Container(api, "FastAPI Gateway", "Python 3.12", "Handles HTTP requests, auth, and enqueues jobs.")
        Container(worker, "Arq Background Worker", "Python 3.12", "Executes heavy PDF parsing and LLM orchestration asynchronously.")
        
        ContainerDb(redis, "Redis", "In-Memory Cache", "Stores raw PDFs temporarily, caches decks, and manages job queues.")
        ContainerDb(postgres, "PostgreSQL", "Relational DB", "Stores user profiles, deck metadata, and encrypted flashcards.")
        ContainerDb(lancedb, "LanceDB", "Vector Store (S3/R2)", "Stores document chunks and embeddings for semantic deduplication.")
    }

    Rel(user, frontend, "Visits", "HTTPS")
    Rel(frontend, api, "POST /pdf/upload", "JSON/Multipart")
    Rel(api, redis, "Enqueues Job & Caches PDF", "RESP")
    Rel(api, postgres, "Creates Document State", "SQL/Asyncpg")
    
    Rel(worker, redis, "Pops Job & Reads PDF", "RESP")
    Rel(worker, postgres, "Updates State & Saves Deck", "SQL/Asyncpg")
    Rel(worker, lancedb, "Stores/Searches Chunks", "Arrow")

```

## 3. Component Diagram (Level 3 - Hexagonal Architecture)

Details the internal structure of the PDF to Anki AI Copilot module, proving the Dependency Inversion Principle.
 
```mermaid

C4Component
    title Component Diagram: PDF to Anki (Hexagonal Architecture)

    Container_Boundary(module, "Module: PDF to Anki AI Copilot") {
        
        Component(router, "FastAPI Router", "Primary Adapter", "Handles HTTP, Auth, and Enqueues to Arq.")
        Component(task, "Arq Task", "Primary Adapter", "Background entry point for process_pdf_task.")
        
        Component(usecase, "Process PDF Use Case", "Application Layer", "Orchestrates parsing, chunking, embedding, and generation.")
        Component(chunking, "Semantic Chunking", "Application Layer", "Splits markdown by headers and paragraphs.")
        
        Component(entities, "Entities & Ports", "Domain Layer", "PdfDeck, PdfFlashcard, PdfParserPort, FlashcardGeneratorPort.")
        
        Component(llamaparse_adapter, "LlamaParse Adapter", "Secondary Adapter", "Implements PdfParserPort.")
        Component(flashcard_adapter, "Flashcard Generator", "Secondary Adapter", "Implements FlashcardGeneratorPort.")
        Component(lancedb_adapter, "LanceDB Adapter", "Secondary Adapter", "Implements VectorStorePort.")
        Component(repo_adapter, "Document Repository", "Secondary Adapter", "Handles ALE encryption and DB persistence.")
    }

    Rel(router, usecase, "Invokes")
    Rel(task, usecase, "Invokes")
    Rel(usecase, chunking, "Uses")
    Rel(usecase, entities, "Depends on abstractions & manipulates state")
    
    Rel(llamaparse_adapter, entities, "Implements", "Dependency Inversion")
    Rel(flashcard_adapter, entities, "Implements", "Dependency Inversion")
    Rel(lancedb_adapter, entities, "Implements", "Dependency Inversion")
    Rel(repo_adapter, entities, "Implements", "Dependency Inversion")
```