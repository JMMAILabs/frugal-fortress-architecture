# ADR 0014: Dynamic RAG Injection & In-Memory Filtering (Nested Learning Loop)
**Date:** 2026-05-12
**Status:** Accepted

## Context
Speech-to-Text models (like Whisper) frequently hallucinate or misspell domain-specific corporate jargon, acronyms, or employee names (e.g., transcribing "FastAPI" as "Fast Happy").
Traditional solutions require fine-tuning the Whisper model, which is economically unviable and operationally complex (MLOps overhead). Furthermore, Enterprise users can accumulate hundreds of glossary rules over time.
 
## Decision
We implemented a "Nested Learning Loop" using Retrieval-Augmented Generation (RAG) to correct the LLM in real-time without fine-tuning, utilizing a dual-strategy based on the compute tier to balance Latency and FinOps:

1. **Explicit Feedback:** When a user sees a mistake in the summary, they reply to the bot with a natural language correction. An internal LLM Judge evaluates it and saves the `(misheard, correction)` pair to PostgreSQL.
2. **Free Tier (In-Memory Deterministic Filtering):** For free users, we use a two-step pipeline (Whisper -> Llama 3). Because we have the transcript *before* summarization, we fetch all active rules from Postgres and compile them into a single, highly efficient Regular Expression automaton (O(N) complexity, sorted by length descending). We filter these rules against the Whisper transcript in-memory (<5ms) and inject *only* the matched rules into the Llama 3 prompt. This guarantees 100% accuracy while saving thousands of input tokens.
3. **Paid Tiers (Single-Pass Multimodal + Product Constraint):** For Premium/Pro/PAYG users, we process audio natively via Vertex AI (Gemini 2.5 Flash). Because we do not have a text transcript prior to the LLM call, we cannot pre-filter rules. To prevent massive context injection from destroying profit margins, we applied a **Product Constraint**: the maximum glossary size for paid tiers is capped at **300 rules**. This allows us to inject the entire active glossary directly into the multimodal prompt in a single pass, maintaining ultra-low latency while strictly bounding FinOps costs to a maximum of ~4,500 tokens per request.

## Consequences
* **Positive:** The system adapts to the user's specific vocabulary instantly, at zero training cost.
* **Positive:** Massive token savings on the Free Tier by avoiding over-injection of irrelevant rules.
* **Positive:** Protects Paid Tier profit margins by capping the maximum glossary size, avoiding the "Denial of Wallet" trap of unbounded RAG, while keeping the architecture simple (no complex LRU eviction logic needed).
* **Positive:** For PAYG users, the cost of injecting the 300 rules is calculated *before* inference and billed directly to their wallet, ensuring 100% margin protection.