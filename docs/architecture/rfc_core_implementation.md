# RFC: Core Implementation — Risk Heatmap, Invariants, Observability

**Status:** Internal RFC  
**Purpose:** Pinpoint high-risk areas, state invariants, and observability injection points before and during core implementation.

---

## 1. Complexity Heatmap

| Area | Risk | Rationale |
|------|------|-----------|
| **Concurrency (asyncio, workers)** | High | Event loop blocking (CPU-bound token count, rerank), shared state, background task cleanup. Mitigation: `asyncio.to_thread()` for CPU-bound; context managers for sessions/locks. |
| **LLM routing (LiteLLM)** | High | Timeouts, context overflow, fallbacks, token counting. Mitigation: tenacity retries, circuit breaker, FinOps callback on every call. |
| **DB transactions** | High | Session lifecycle, connection pool exhaustion, advisory locks. Mitigation: `async with sessionmanager.session()`, statement_timeout, retry on OperationalError. |
| **Redis (cache, rate limit, queues)** | Medium | Serialization, Lua scripts, cluster CROSSSLOT. Mitigation: single-key or hash-tag keys; timeouts on all ops. |
| **LanceDB (vector_store)** | Medium | S3/R2 credentials, optional dependency. Mitigation: `connect_lancedb_async()` returns None when URI empty or lancedb missing; callers handle gracefully. |
| **Config (pydantic-settings)** | Low | Validation at startup; secrets via SecretStr. |
| **Telemetry / FinOps logging** | Low | Fire-and-forget; no critical path. |

---

## 2. State Invariants

The following must hold regardless of refactors:

1. **Token usage:** Every LLM request (success or failure) MUST be logged with `model_name`, `input_tokens`, `output_tokens`, and `estimated_cost` (or equivalent) before the response is returned or the flow ends. No silent LLM call.

2. **Tenant isolation:** Every feedback log and every cost/budget record MUST have an associated `tenant_id` (or "system" when not in request context). Never log or persist without tenant context.

3. **Sessions:** Every DB session MUST be acquired via context manager and closed in a `finally` block (or equivalent). No leaking sessions.

4. **Idempotency:** Background jobs that perform external side effects (e.g. LLM call, write to DB) MUST be idempotent (e.g. keyed by input hash) so that retries do not double-charge or double-write.

5. **Secrets:** No API keys or secrets in log messages or exception messages. PII redacted before any text is sent to LLM or written to logs.

---

## 3. Observability Map

Structured logging (JSON) and OpenTelemetry trace IDs MUST be present at these points:

| Location | What to log / inject |
|----------|----------------------|
| **Request entry (middleware)** | `trace_id`, `request_id`, `path`, `method`. Set trace_id in context vars for downstream. |
| **DB session / query** | Already: SQL comment with `trace_id`, `tenant_id`. Keep. |
| **Redis ops** | Optional: trace_id in log context for cache get/set on miss/hit. |
| **LLM request start** | `trace_id`, `tenant_id`, `model` in metadata; FinOpsCallback logs success/failure with these. |
| **FinOpsCallback** | `finops_transaction` / `finops_transaction_failure` with model, tokens, cost, latency_ms, tenant_id, trace_id. |
| **Telemetry POST /log** | `telemetry_event` with project_id, event_type, ip_hash, meta (no raw IP). |
| **Vector store (LanceDB)** | On connect: `lancedb_connected` with truncated URI; on failure: `lancedb_connect_failed`. |
| **Background job start/end** | job_id, task_name, duration, success/failure. |
| **Circuit breaker state change** | State (open/half-open/closed), key. |

Existing `structlog` and `setup_observability` (OpenTelemetry) already cover request instrumentation and LiteLLM. This RFC does not require new libraries; ensure the above keys are present in the existing log and trace pipeline.

---

## 4. Verification

- **Unit tests:** Config validation, `log_token_usage()` output, `connect_lancedb_async()` with empty URI returns None.
- **Integration tests:** One request through a route that calls LLM; assert one `finops_transaction` (or equivalent) log line with tokens and cost. DB session tests with cleanup.
- **Edge cases:** LanceDB missing (optional dep), Redis down (graceful degradation or fail-fast per design), DB timeout (retry then fail).
