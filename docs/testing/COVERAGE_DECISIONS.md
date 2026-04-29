# Coverage Decisions and Uncovered Lines

This document records **intentional** and **pragmatic** decisions about test coverage: what was added, what was not covered, and why. It avoids vanity metrics and diminishing returns.

---

## 1. Summary of Coverage Improvements

### Previous passes
| Target | Change | Tests Added |
|--------|--------|-------------|
| `app.core.finops_logger` | 0% → 100% | `tests/unit/test_finops_logger.py` |
| `app.core.config` | lancedb_storage_options | `test_core_config_lancedb_storage_options_all_branches` in `test_coverage_boost.py` |
| `app.core.feature_flags` | L1 expiry + _prune_l1 | `test_feature_flags_l1_expired_entry_evicted`, `test_feature_flags_prune_l1_when_over_maxsize` |
| `app.core.retrieval` | available_tokens < 0 | `test_retrieval_execute_prompt_too_long_context_empty` |
| `app.infrastructure.search` | empty queries | `test_search_engine_empty_queries_returns_empty` |
| `app.infrastructure.queue` | line 43 double-check | `test_queue_get_pool_double_check_second_caller_gets_pool` |
| `app.dependencies` | line 158 else raise | `test_dependencies_budget_manager_raises_when_not_counter` |
| `app.infrastructure.observability` | metrics endpoint | `test_observability_metrics_endpoint_hit` |

### 2026-03-15 pass (max coverage without vanity / diminishing returns)
| Target | Change | Tests Added |
|--------|--------|-------------|
| `app.core.config` | 88% → 95% (CORS prod, select_fast_model, override_database_url) | `test_core_config_cors_*`, `test_core_config_select_fast_model_*`, `test_core_config_override_database_url_*` in `test_coverage_boost.py` |
| `app.core.telemetry` | 85% → 100% | `test_telemetry_attention_endpoint_success`, `test_telemetry_attention_add_task_raises_logs_warning` in `test_coverage_boost.py` |
| `app.core.vector_store` | 36% → 91% | `tests/unit/test_core_vector_store.py` (ImportError path, connect_async success) |
| `app.core.prompts` | 93% → 96% (L1 expiry, _prune_l1) | `test_prompts_l1_expired_entry_evicted`, `test_prompts_prune_l1_when_over_maxsize` in `test_coverage_boost.py` |
| `app.modules.audio_notes.application.process_feedback` | 80% → 96% (overwrite inactive) | `test_process_feedback_storage_limit_overwrite_inactive` in `test_audio_notes_application.py` |
| `app.modules.audio_notes.infrastructure.idempotency` | 74% → 94% (get_cached dict/str/JSON) | `test_idempotency_get_cached_when_cache_returns_*` in `test_audio_notes_module.py` |

---

## 2. Lines / Branches Not Covered and Why

### 2.1 Modules at 0% or Very Low Coverage (By Design / External Dependency)

| Module | Coverage | Reason Not Covered |
|--------|----------|--------------------|
| **`app.core.finops_logger`** | Now 100% | Was 0%; now fully covered by `test_finops_logger.py`. |
| **`app.modules.pdf_anki.infrastructure.embedding_adapter`** | 0% | Thin adapter over external embedding API; covered by contract/E2E when the feature is used. Unit tests would require heavy mocking of the external SDK. |
| **`app.modules.pdf_anki.infrastructure.flashcard_generator`** | 0% | LLM-dependent flashcard generation; same as above. |
| **`app.modules.pdf_anki.infrastructure.llamaparse_adapter`** | 0% | LlamaParse HTTP API adapter; requires API key and network. E2E or contract tests when PDF-Anki is in scope. |
| **`app.modules.pdf_anki.infrastructure.lancedb_adapter`** | 25% | LanceDB/S3 integration; requires real or testcontainers for meaningful coverage. Current tests cover the interface; deep coverage deferred. |
| **`app.modules.pdf_anki.infrastructure.document_repository`** | 30% | DB + file storage; integration tests cover main paths. Remaining lines are error/edge paths. |
| **`app.modules.audio_notes.infrastructure.whisper_groq`** | 42% | Groq Whisper API; external service. |
| **`app.modules.audio_notes.infrastructure.repositories`** | 44% | DB and storage; integration tests preferred over unit mocks. |
| **`app.modules.audio_notes.infrastructure.summarizer`** | 46% | LLM summarization; external dependency. |
| **`app.core.vector_store`** | 91% | Remaining: lines 26, 33 (_get_connection_type cache when lancedb already loaded). |

**Principle:** Adapters that are thin wrappers around third-party or external services are not forced to 100% unit coverage; integration or E2E tests and resilience (retries, circuit breakers) are preferred.

---

### 2.2 Branches / Single Lines Not Covered (Pragmatic)

- **`app.core.concurrency`** (97%): Branches `54→exit`, `88→97`, `121→132`, `181→183`, `263→265` are rare error or shutdown paths; covered by integration/concurrency tests where feasible.
- **`app.core.router`** (98%): Branches `559→581`, `574→581`, `664→667`, `676-677`, `698→700` — e.g. `TypeError` when stream is not async-iterable, or `ctx is None` in error handlers. Edge cases; high mock complexity for low ROI.
- **`app.core.prompts`** (96%): Lines 83, 123 remain (L1 eviction edge); L1 expiry and _prune_l1 loop now covered.
- **`app.infrastructure.cache`** (91%): Many branches (47, 89-90, 190-192, 235, 239-240, etc.) are error paths, compression edge cases, or Redis failure handling; partially covered by integration tests.
- **`app.infrastructure.llm_gateway`** (96%): Lines 211, 351→exit, 376→378, 389→368, 406-407, 432 — mid-stream failure, retry exhaustion, or provider-specific branches. Covered by existing resilience tests; remaining are narrow branches.
- **`app.infrastructure.database`** (98%): Lines 132, 207-208 — “No such event” debug path and `create_all`; low impact.
- **`app.main`** (95%): Lines 60-61 (Langfuse env), 65→84 (lifespan init when not TESTING), 91→exit (shutdown exception). Startup/shutdown paths are tested via TestClient and integration; env-specific branches are optional.
- **`app.middleware.audit`** (91%): 54-56 (`http.disconnect`), 133→exit (exception re-raise). Disconnect and error paths are edge cases.
- **`app.worker`** (85%): Lines 212-218, 312→314, 318-321, 373→exit, 379→376, etc. Worker loops and failure branches are covered by integration tests (`test_worker_*.py`); remaining are granular branches.

**Principle:** We do not add tests for every single branch when the cost (flaky mocks, maintenance) outweighs the benefit. We prefer integration and resilience tests for I/O and failure paths.

---

### 2.3 Circuit Breaker, FinOps Callback, Reranker (99% / 97%)

- **`app.infrastructure.circuit_breaker`** (99%): Line 237 — half-open probe path; covered by circuit-breaker-specific tests.
- **`app.infrastructure.finops_callback`** (97%): Branch 28→37; callback branch.
- **`app.infrastructure.reranker_adapter`** (99%): Branch 46→48; internal branch.

These are left as-is; raising to 100% would require very narrow mocks for minimal gain.

---

## 3. How to Run Coverage

```bash
# Full suite with coverage (as in CI/make check)
make check

# Or explicitly
uv run pytest tests -n 6 --cov=src --cov-report=term-missing --cov-report=html:reports/coverage_html --cov-report=xml:reports/coverage.xml
```

---

## 4. Policy

- **Target:** Maintain or improve overall coverage (e.g. ~90%+) without chasing 100% on every file.
- **Prioritise:** Domain logic, security (guardrails, PII, auth), resilience (circuit breakers, retries), and FinOps/token logging.
- **Accept:** Lower or no unit coverage on thin external adapters (LlamaParse, Whisper, LanceDB S3, etc.) when integration or E2E coverage exists or is planned.
- **Document:** Any new “by design” uncovered area should be added to this file.

---

*Last updated: 2026-03-15 (coverage boost: config, telemetry 100%, vector_store, prompts, process_feedback, idempotency). Total ~88%.*
