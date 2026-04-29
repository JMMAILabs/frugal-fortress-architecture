# DSPy Pilots: Compiled Heuristic Policies

## 1. Overview
The Frugal Fortress backend supports optional DSPy pilots. Instead of relying on brittle, hand-crafted prompts and hardcoded heuristics, we use DSPy to compile and optimize routing policies and system prompts offline. The backend then loads these compiled JSON artifacts at runtime.

This approach allows us to improve AI accuracy and reduce latency without altering the underlying HTTP contracts or core Python domain logic.

## 2. Active Pilots

### A. AdaptiveRouter Policy (`config/dspy/adaptive_router.json`)
When enabled, this artifact replaces the standard LLM-based classifier and query expander.
*   **Impact:** Skips auxiliary LLM calls (`classifier`, `query_expansion`), reducing latency by ~150ms and saving input tokens.
*   **Mechanism:** Uses a compiled heuristic policy to classify prompt complexity and generate search query variants locally.
*   **Feature Flags:** `DSPY_ENABLED`, `DSPY_ADAPTIVE_ROUTER_ENABLED`, `DSPY_ADAPTIVE_ROUTER_ROLLOUT_PERCENT`.

### B. PDF-Anki Prompt Optimization (`config/dspy/pdf_anki.json`)
When enabled, this artifact dynamically injects optimized instructions into the base prompt based on the user's tier and the selected pipeline (Chunked vs. Unified).
*   **Impact:** Improves flashcard alignment scores and reduces duplicate generation.
*   **Mechanism:** Preserves the existing dynamic Few-Shot injection (Nested Learning Loop) but enhances the static system instructions.
*   **Feature Flags:** `DSPY_ENABLED`, `DSPY_PDF_ANKI_ENABLED`, `DSPY_PDF_ANKI_ROLLOUT_PERCENT`.

## 3. Offline Compilation Workflow
We do not run DSPy optimizers in production. The workflow is strictly offline:

1. **Export Real Datasets:**
   Extract real user interactions from the PostgreSQL `feedback_log` or Arize Phoenix traces.
   ```bash
   PYTHONPATH=src python scripts/export_adaptive_router_dspy_dataset.py \
     --source db --since-days 30 \
     --output config/dspy/examples/adaptive_router_real_export.jsonl