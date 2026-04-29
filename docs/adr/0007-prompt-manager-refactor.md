# 6. Prompt Manager Refactor (Validation & Resilience)
**Date:** 2026-02-09
**Status:** Accepted
## Context
The initial `PromptManager` implementation blindly rendered Jinja2 templates fetched from the database. This posed two risks:
1.  **Data Integrity:** If the code failed to pass a required variable (e.g., `{{ context }}`), the prompt would render with an empty string, causing the LLM to hallucinate or fail silently.
2.  **Resilience:** Reliance solely on DB/Redis meant that infrastructure outages could break the core AI functionality.
## Decision
1.  **Input Validation:** We extended `PromptManager` to check `kwargs` against the `input_variables` column in the `PromptTemplate` entity. Missing variables now trigger a structured warning log (and can be configured to raise exceptions in strict mode).
2.  **Fallback Strategy:** We maintain a hardcoded `FALLBACK_TEMPLATES` dictionary in code. This ensures that even if the Database and Redis are both down, critical flows (like RAG and Classification) continue to function with baseline prompts.
3.  **Caching:** We continue to use Redis for caching templates (TTL 5 mins) to reduce DB load.
## Consequences
*   **Positive:** Higher reliability. Developers get immediate feedback via logs if they mismatch prompt variables.
*   **Negative:** Slight overhead in parsing `input_variables` string.
*   **Future Work:** Move `FALLBACK_TEMPLATES` to a local JSON file to allow updates without code deployment (K8s ConfigMap pattern).
