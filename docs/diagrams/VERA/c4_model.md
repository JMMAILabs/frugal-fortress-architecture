# C4 Model: Receipt Parser Module

This document provides the C4 Model diagrams for the Receipt Parser module, highlighting Edge Compute offloading and the Human-in-the-Loop (HITL) architecture.

## 1. System Context (Level 1)

```mermaid
C4Context
    title System Context: Receipt Parser (Frugal Fortress)

    Person(user, "Employee / Accountant", "Captures receipts via mobile PWA.")
    System(receipt_parser, "Receipt Parser System", "Validates math, extracts structured JSON, and learns from corrections.")
    
    System_Ext(llamaparse, "LlamaParse API", "Extracts markdown from receipts (Free Tier).")
    System_Ext(vertex_ai, "Google Vertex AI", "Vision-Language Model extraction (Premium/Pro/PAYG).")
    System_Ext(groq, "Groq API", "Fast JSON extraction from markdown (Free Tier).")

    Rel(user, receipt_parser, "Uploads compressed image & corrects math", "HTTPS")
    Rel(receipt_parser, llamaparse, "Sends Image, gets Markdown", "HTTPS/REST")
    Rel(receipt_parser, vertex_ai, "Sends Image, gets JSON", "HTTPS/gRPC")
    Rel(receipt_parser, groq, "Sends Markdown, gets JSON", "HTTPS/REST")

```

## 2. Container Diagram (Level 2)
```mermaid

C4Container
    title Container Diagram: Receipt Parser

    Person(user, "User", "Uses Mobile PWA")
    
    Container(pwa, "Next.js PWA (Edge)", "React, Canvas API", "Compresses images locally (Max 1024px, JPEG 0.8) to save bandwidth and VLM tokens. Handles offline queue via IndexedDB.")
    
    Container_Boundary(vpc, "Frugal Fortress VPC") {
        Container(api, "FastAPI Gateway", "Python 3.12", "Handles HTTP requests, PII scrubbing, and LLM orchestration.")
        
        ContainerDb(redis, "Redis", "In-Memory Cache", "Handles Token Bucket Rate Limiting and FinOps Budgeting.")
        ContainerDb(postgres, "PostgreSQL", "Relational DB", "Stores encrypted receipts (ALE) and vendor embeddings (pgvector).")
    }

    Rel(user, pwa, "Captures Photo")
    Rel(pwa, api, "POST /receipts/process (Base64)", "JSON/HTTPS")
    Rel(api, redis, "Check Budget & Rate Limits", "Lua Script")
    Rel(api, postgres, "Save Verified Receipt", "SQL/Asyncpg")
```

## 3. Component Diagram (Level 3 - Hexagonal Architecture)
```mermaid
 
C4Component
    title Component Diagram: Receipt Parser (Hexagonal Architecture)

    Container_Boundary(module, "Module: receipt_parser") {
        
        Component(router, "FastAPI Router", "Primary Adapter", "Exposes /process and /history endpoints.")
        
        Component(processor, "Receipt Processor", "Application Layer", "Orchestrates the Self-Correction Loop and Tier Routing.")
        Component(math_validator, "Math Validator", "Domain Logic", "Pure Python deterministic math validation (Subtotal + Tax = Total).")
        
        Component(schemas, "Receipt Schemas", "Domain Layer", "Strict Pydantic models for JSON validation.")
        
        Component(pii_scrubber, "PII Scrubber", "Security", "Microsoft Presidio + Regex to redact sensitive data before LLM.")
        Component(repo_adapter, "Receipt Repository", "Secondary Adapter", "Handles ALE encryption and pgvector Few-Shot retrieval.")
    }

    Rel(router, processor, "Invokes")
    Rel(processor, pii_scrubber, "Sanitizes OCR text (Free Tier)")
    Rel(processor, math_validator, "Validates LLM JSON output")
    Rel(processor, schemas, "Enforces structure")
    Rel(processor, repo_adapter, "Persists data & fetches Few-Shot examples")

```