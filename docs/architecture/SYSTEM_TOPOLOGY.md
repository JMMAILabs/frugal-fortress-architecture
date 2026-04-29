# System Topology & Ecosystem

The "Frugal Fortress" ecosystem is designed to push compute to the Edge whenever possible, reserving the Cloud solely for secure, heavy orchestration, data persistence, and observability.

## 1. Repository Inventory

The core ecosystem is strictly divided into 2 repositories to enforce separation of concerns between the delivery mechanism and the business logic:

1.  📦 **`backend-monolith` (Python/FastAPI):** The secure core. Handles Audio Notes, Receipt Parsing, PDF-Anki, and GenUI orchestration. Deployed in a secure, containerized Virtual Private Cloud (VPC).
2.  💻 **`frontend-platform` (Next.js):** The interactive delivery layer. Contains the UIs for GenUI, Receipt Parser, and PDF-Anki. Deployed globally via Vercel's Edge Network.
 
*Note: Other isolated edge applications may ping the backend's telemetry endpoints (`/api/v1/log`) for centralized observability (Grafana/Loki), but they are not considered part of this core product ecosystem.*

## 2. Network & Deployment Topology

```text
[ END USER ]
    │
    └── Access via Web Browser / PWA ────────────────────┐
                                                         │
        [ Vercel Cloud ] (Serverless / Edge CDN)         │
        ┌──────────────────────────────────────────┐     │
        │ REPO 2: frontend-platform (Next.js)      │     │
        │  ├─ /genui                               │     │
        │  ├─ /pdf-anki                            │     │
        │  └─ /receipts (PWA)                      │     │
        └────────────────────┬─────────────────────┘     │
                             │ (HTTPS / JSON / SSE)      │
                             ▼                           │
        [ Secure VPC ] (Containerized Cluster)           │
        ┌──────────────────────────────────────────┐     │
        │ REPO 1: backend-monolith (Python)        │◄────┘
        │  ├─ FastAPI (API Gateway & Router)       │
        │  ├─ Audio Notes Domain                   │
        │  ├─ PDF to Anki Domain                   │
        │  ├─ Receipt Parser Domain                │
        │  ├─ GenUI Domain                         │
        │  └─ DLP Proxy (Security Guardrail)       │
        └──────────────────────────────────────────┘
```

## 3. Edge vs. Cloud Compute Distribution

To maintain the $5/month infrastructure baseline, compute is strictly distributed:

*   **Edge Compute (Frontend):** 
    *   Image resizing and compression (Canvas API) for receipts.
    *   Static UI rendering and caching.
    *   Client-side form validation.
*   **Cloud Compute (Backend):** 
    *   PII Scrubbing (Local SLM / Presidio).
    *   Vector Search (pgvector / LanceDB).
    *   LLM Orchestration and Self-Correction Loops.
    *   Application-Level Encryption (ALE).