# KERA: Knowledge Extraction & Retention Architecture

**KERA** is the Async Knowledge Beast of the Frugal Fortress. It is a heavy-compute, asynchronous pipeline designed to ingest massive corporate documents (PDFs) and generate high-signal, spaced-repetition flashcards.

Built for high concurrency, KERA ensures the FastAPI Event Loop is never blocked by CPU-bound tasks, absorbing massive traffic spikes gracefully.

## Core Capabilities

*   **Async Burst Ingestion:** Uploads return an HTTP 202 Accepted instantly. Heavy parsing is offloaded to isolated `Arq` process pools (`CpuBoundExecutor`), preventing database connection starvation.
*   **Dual-Pipeline Processing:** 
    *   *Free/Admin Tiers:* LlamaParse extraction -> Semantic Chunking -> LanceDB Vector Search -> Groq (Llama-3.3).
    *   *Paid Tiers:* Direct multimodal ingest via Google Vertex AI (Gemini 2.5 Flash) ensuring Zero Data Retention.
*   **Multi-Tier Caching:** 
    *   **L1 (Exact Hash):** Bypasses processing entirely for duplicate PDFs.
    *   **L2 (Semantic Deduplication):** Uses LanceDB to skip LLM generation for chunks that are >95% similar to previously processed concepts.
*   **DSPy Prompt Optimization:** Heuristic policies compiled offline dynamically adjust the System Prompt based on the user's tier, optimizing token usage and study quality without altering runtime logic.

## Deep Dives

Explore the specific architectural implementations for this module:

*   [FinOps, PAYG Wallet & Dynamic Billing](PAYG_WALLET_AND_BILLING.md)
*   [Capacity Planning & Cost Matrix](finops_capacity_planning.md)
*   [Native Support Tickets Architecture](SUPPORT_TICKETS_ARCHITECTURE.md)