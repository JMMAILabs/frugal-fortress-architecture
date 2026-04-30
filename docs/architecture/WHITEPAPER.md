# Architecture Whitepaper: The Frugal Fortress

**Domain:** Enterprise AI, FinOps, Site Reliability Engineering (SRE)
**Architecture:** Modular Monolith (Strict Hexagonal / Ports & Adapters)

## 1. The Challenge
Building a heavy document processing system (e.g., PDFs to AI-generated study flashcards, OCR receipt parsing) that scales to thousands of concurrent users presents two major hurdles:
1.  **Economic Viability:** Naive LLM API usage destroys profit margins.
2.  **Compliance:** Processing sensitive data (Invoices, Medical PDFs) requires strict adherence to SOC2 and Zero Data Retention policies.

## 2. The Architecture: Modular Monolith over Microservices
We deliberately chose a **Modular Monolith** (Python/FastAPI) over premature microservices. This eliminates network latency between domains, simplifies deployments, and maximizes resource sharing (Database connection pools, Redis multiplexing).

We enforce a strict **Hexagonal Architecture**. The core domain (e.g., `process_pdf_use_case`) is completely agnostic to the underlying infrastructure. Whether the LLM provider is Groq (free tier) or Google Vertex AI (paid tiers), the business logic remains pure and untouched.

## 3. Resilience Patterns (SRE)
In AI, third-party providers *will* fail. A naive application collapses; a resilient system adapts.
*   **Distributed Circuit Breakers:** Implemented a Redis-backed circuit breaker. If the primary LLM provider experiences a rate limit (HTTP 429) or outage, the circuit opens, automatically triggering a "Graceful Degradation" fallback to a cheaper/faster secondary model. The user never sees a 500 error.
*   **Idempotent Background Workers:** Heavy parsing is offloaded to `Arq` workers. We implemented strict idempotency keys based on the `SHA-256` hash of the file bytes. If a worker crashes or a user double-clicks, the system guarantees the file is only processed—and billed—once.

## 4. Innovation: The "Nested Learning Loop"
Fine-tuning LLMs is slow and expensive. Instead, we built a self-correcting RAG system.
When a user edits an AI-generated flashcard or corrects a receipt to fix a formatting or factual error, the backend vectorizes this correction and stores it in `pgvector` as an approved 'Few-Shot' example. The next time that user uploads a document, the system dynamically injects their past corrections into the System Prompt. **The system learns the user's style in real-time, at zero training cost.**

## 5. Zero-Compromise Testing
To guarantee enterprise reliability, standard unit tests are not enough.
*   **Mutation Testing:** We integrated `mutmut` to actively modify the source code during CI/CD, ensuring our test suite actually catches regressions, preventing "vanity coverage" metrics.
*   **LLM-as-a-Judge:** We integrated `DeepEval` for semantic evaluation, ensuring the quality of the generated outputs does not degrade when we swap underlying LLM models.  