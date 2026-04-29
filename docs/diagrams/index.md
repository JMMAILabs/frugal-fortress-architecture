# Architectural Diagrams Index

Visual representations of the **Frugal Fortress** architecture, designed for Enterprise Architects, SREs, and Security Compliance auditors. 

The structure of these diagrams mirrors the codebase structure, divided into global cross-cutting concerns (Core) and isolated product domains (Modules).

---

## 1. Core Architecture & Infrastructure
Cross-cutting concerns that apply to the entire monolith and infrastructure stack.

| Diagram | Audience | Description |
| :--- | :--- | :--- |
| **[System Context (C4)](core/c4_system_context.md)** | CTO, Architects | System boundaries, external integrations (LLMs, Auth), user interactions. |
| **[Backend Monolith Context](core/backend_monolith_context.md)** | Architects | High-level: Core, Shared, Modules, API routes, external services. |
| **[Infrastructure & Deployment](core/infrastructure_view.md)** | DevOps, SRE | Docker/Kubernetes, networking, observability (OTEL/Grafana). |
| **[Database Schema](core/database_schema.md)** | DBAs | Entity relationships, partitioning for `FeedbackLog`, vector indexing. |
| **[FinOps & Token Lifecycle](core/finops_lifecycle.md)** | FinOps, Product | Budget reservation and rate limiting (two-phase commit via Lua). |
| **[Concurrency & SingleFlight](core/concurrency_singleflight.md)** | Backend | Cache stampede prevention under viral traffic. |
| **[Circuit Breaker State Machine](core/circuit_breaker_state.md)** | SRE, DevOps | Distributed and local fallback for external dependencies. |
| **[Resilient Learning Loop](core/resilient_learning_loop.md)** | Backend, Architects | Request/Response lifecycle, caching, async persistence. |
| **[RAG & AI Pipeline](core/rag_pipeline.md)** | AI Engineers | Hybrid search (vector + keyword), reranking, feedback loop. |
| **[Hexagonal Module Template](core/hexagonal_module_template.md)** | Backend Engineers | Standard Ports & Adapters template used across all modules. |

---

## 2. Bounded Contexts (Modules)
Specific business logic, frontend interactions, and data flows for each independent product module.

### AURA (Audio Notes)
| Diagram | Description |
| :--- | :--- |
| **[C4 Model](AURA/c4_model.md)** | Context, Container, and Component views for Audio Notes. |
| **[Idempotency & Learning Flow](AURA/diagram_flow.md)** | Hash → Redis check → transcribe/summarize → LLM Judge correction. |
| **[PAYG Billing Lifecycle](AURA/diagram_payg_billing.md)** | Dynamic model selection and Two-Phase Commit (2PC) wallet deductions. |
| **[Support Tickets & Debouncing](AURA/diagram_support_tickets.md)** | Redis Media Group debouncing for concurrent Telegram image uploads. |
| **[Glossary Management](AURA/diagram_glossary_management.md)** | Inline keyboard pagination and XLSX export flow. |

### KERA (PDF to Anki)
| Diagram | Description |
| :--- | :--- |
| **[C4 Model](KERA/c4_model.md)** | Context, Container, and Component views for PDF to Anki. |
| **[Async RAG Flow](KERA/diagram_pdf_anki.md)** | Upload → worker → cache → LlamaParse/Vertex → vector search → generation. |
| **[Concurrent Polling & Telemetry](KERA/diagram_frontend_polling.md)** | Next.js non-blocking task polling and viewport attention tracking. |

### VERA (Receipt Parser)
| Diagram | Description |
| :--- | :--- |
| **[C4 Model](VERA/c4_model.md)** | Context, Container, and Component views for the Receipt Parser. |
| **[Hexagonal Architecture](VERA/diagram_hexagonal_module.md)** | Internal wiring of ports, adapters, and the pure Python Math Validator. |
| **[Sequence: Edge & HITL](VERA/sequence_receipts.md)** | Self-correction loop: Edge compression, PII redaction, math validation, retries. |
| **[Response Contract](VERA/diagram_response_contract.md)** | HTTP 200 vs HTTP 206 (Human-in-the-Loop) decision tree. |
| **[History & Decryption Flow](VERA/diagram_history_flow.md)** | Application-Level Encryption (ALE) decryption on the fly. |
| **[Export CSV Flow](VERA/diagram_export_csv.md)** | In-memory CSV generation from decrypted database records. |
| **[Frontend Offline Sync](VERA/diagram_frontend_offline_sync.md)** | PWA IndexedDB queueing and state recovery during network drops. |