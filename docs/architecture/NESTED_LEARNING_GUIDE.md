# Nested Learning Architecture Guide

## 1. Overview
The "Frugal Fortress" implements a **Nested Learning Loop** (Self-Improving AI) across all its modules. Unlike static RAG (Retrieval-Augmented Generation) systems, this architecture learns from explicit user feedback in real-time. It dynamically adjusts prompts using **Few-Shot Injection** without requiring expensive and slow model fine-tuning cycles.

## 2. Module Implementations
 
### A. Audio Notes: The Dynamic Glossary
*   **The Problem:** Whisper often mishears domain-specific jargon (e.g., "FastAPI" transcribed as "Fast Happy").
*   **The Loop:** 
    1. User replies to the bot with a correction: *"It's FastAPI, not Fast Happy"*.
    2. An LLM Judge evaluates the correction and extracts the pair `{"misheard": "Fast Happy", "correction": "FastAPI"}`.
    3. This is saved to the user's `GlossaryRule` table in Postgres.
    4. On the next audio upload, the backend fetches active rules from Redis and injects them into the LLM Summarizer's System Prompt. The AI learns the user's vocabulary instantly.

### B. Receipt Parser: Vendor-Specific Few-Shot
*   **The Problem:** Obscure vendors have weird receipt layouts that confuse Vision-Language Models (VLMs).
*   **The Loop:**
    1. The Math Validator detects an error (e.g., Subtotal + Tax != Total) and returns HTTP 206.
    2. The user manually corrects the JSON in the UI and saves it.
    3. The backend generates an embedding of the `issuer_name` (e.g., "Uber") and saves the corrected JSON to Postgres.
    4. Next time a receipt is uploaded, the system performs a `pgvector` cosine similarity search on the `issuer_name`. If a match is found, the previously corrected JSON is injected as a `<example>` block in the prompt. The system adapts to new receipt formats automatically.

### C. PDF to Anki: Positive/Negative Reinforcement
*   **The Problem:** The AI generates flashcards that are too broad or redundant.
*   **The Loop:**
    1. The user reviews the deck and clicks "Edit" (to improve a card) or "Discard" (to delete a bad card).
    2. The backend vectorizes the chunk of text that generated the card and saves the action to `FeedbackLog`.
    3. During future PDF processing, the worker searches `LanceDB` for similar chunks. 
    4. Edited cards are injected as **Positive Few-Shot** (`"Format your answers like this:"`). Discarded cards are injected as **Negative Few-Shot** (`"DO NOT generate cards like this:"`).

## 3. The Core Engine: `FeedbackLog` & `pgvector`
At the heart of this system is the `FeedbackLog` entity (`src/app/domain/feedback.py`).
*   **Storage:** PostgreSQL + `pgvector`.
*   **Partitioning:** To maintain sub-millisecond retrieval times as the dataset grows, the table is partitioned by month (`RANGE (created_at)`).
*   **Hybrid Search:** The `AdaptiveRouter` uses a combination of Vector Cosine Distance (`<=>`) and Keyword Search (`TSVECTOR`), merged via Reciprocal Rank Fusion (RRF), to find the most relevant past interactions.