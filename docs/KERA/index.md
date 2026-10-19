# KERA: Knowledge Extraction & Retention Architecture

**KERA** is the Async Knowledge Beast of the Frugal Fortress. It is a heavy-compute, asynchronous pipeline designed to ingest massive corporate documents (PDFs) and generate high-signal, spaced-repetition flashcards.

Built for high concurrency, KERA ensures the FastAPI Event Loop is never blocked by CPU-bound tasks, absorbing massive traffic spikes gracefully.

## Core Capabilities

*   **Async Burst Ingestion:** Uploads return an HTTP 202 Accepted instantly. Heavy parsing is offloaded to isolated `Arq` process pools (`CpuBoundExecutor`), preventing database connection starvation.
*   **Dual-Pipeline Processing:** 
    *   *Free/Admin Tiers:* LlamaParse extraction -> Semantic Chunking -> LanceDB Vector Search -> Groq (GPT-OSS 120B).
    *   *Paid Tiers:* Direct multimodal ingest via Google Vertex AI (Premium: Gemini 3.1 Flash-Lite; Pro: Gemini 3.5 Flash-Lite) ensuring Zero Data Retention. PAYG also offers Gemini 3.8 Flash; see [ADR-0017](../adr/0017-gemini-models-25-migration.md).
*   **Multi-Tier Caching:** 
    *   **L1 (Exact Hash):** Bypasses processing entirely for duplicate PDFs.
    *   **L2 (Semantic Deduplication):** Uses LanceDB to skip LLM generation for chunks that are >95% similar to previously processed concepts.
*   **DSPy Prompt Optimization:** Heuristic policies compiled offline dynamically adjust the System Prompt based on the user's tier, optimizing token usage and study quality without altering runtime logic.

## Deep Dives

Explore the specific architectural implementations for this module:

*   [FinOps, PAYG Wallet & Dynamic Billing](PAYG_WALLET_AND_BILLING.md)
*   [Capacity Planning & Cost Matrix](finops_capacity_planning.md)
*   [Native Support Tickets Architecture](SUPPORT_TICKETS_ARCHITECTURE.md)