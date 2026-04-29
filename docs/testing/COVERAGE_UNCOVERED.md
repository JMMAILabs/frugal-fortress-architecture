# Intentional Coverage Exceptions

This document justifies the lines or branches that are **intentionally excluded** from unit test coverage. This aligns with **ADR-0003**, which explicitly rejects "Vanity Metrics" in favor of system stability and integration testing.

**Current Baseline Target:** 95%

## 1. Excluded Modules (Facades & Thin Adapters)
These files are omitted in `pyproject.toml` (`[tool.coverage.run] omit`) because they are either thin wrappers around third-party SDKs or simple re-exports. Testing them would require deep, brittle mocking of external services.

| File | Reason for Exclusion |
| :--- | :--- |
| `app.core.database.py` | Re-export of `infrastructure.database`. |
| `app.modules.pdf_anki.infrastructure.llamaparse_adapter.py` | Thin wrapper over LlamaIndex SaaS. Validated via E2E tests. |
| `app.modules.pdf_anki.infrastructure.lancedb_adapter.py` | S3/R2 Vector DB integration. Validated via Integration tests. |
| `app.modules.audio_notes.infrastructure.whisper_groq.py` | External Groq Whisper API. Requires real audio bytes to test meaningfully. |

## 2. Pragmatic Branch Exceptions
We do not write tests for highly improbable edge cases if the cost of mocking outweighs the benefit.

*   **`app.core.concurrency`:** OS-specific fallback branches (e.g., `sched_getaffinity` failures on macOS/Windows).
*   **`app.infrastructure.cache`:** Catastrophic Redis driver failures (e.g., `CROSSSLOT` errors during Lua script execution). These are handled defensively but are difficult to trigger deterministically in CI.
*   **`app.infrastructure.llm_gateway`:** Mid-stream chunk decoding failures from specific LLM providers. Covered by general circuit breaker resilience tests.
*   **`app.main`:** ASGI Lifespan startup/shutdown exception blocks.

## 3. Policy for New Exceptions
1. **Prioritize:** Domain logic, Security (PII Scrubbing, Auth), Resilience (Circuit Breakers), and FinOps (Token counting). These must remain near 100%.
2. **Accept:** Lower unit coverage on external I/O adapters, provided they are covered by Contract or Integration tests.
3. **Document:** Any new intentional gap must be recorded in this file. Do not lower the global 95% threshold.