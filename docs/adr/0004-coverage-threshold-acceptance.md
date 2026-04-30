# ADR 0004: Acceptance of 90%+ Coverage Baseline

**Date:** 2026-01-31
**Status:** Accepted (Canonical threshold; supersedes [ADR-0003](0003-coverage-strategy-asyncio.md))

## Context
The project sustains a coverage level **at or above 90%** across the entire codebase. The remaining uncovered lines have been audited and identified as:

1.  **Infrastructure Resilience (Defensive Coding):**
    *   `src/app/infrastructure/llm_gateway.py`: Specific vendor branches (Groq / Vertex AI) and deep exception handling for local embedding generation that are covered by logic tests but missed by the tracer due to async context switching.
    *   `src/app/infrastructure/cache.py`: Catastrophic Redis driver failures (e.g., `ConnectionError` during pipeline execution) which are mocked but difficult to hit deterministically in high-concurrency async tests.

2.  **OS-Specific Concurrency:**
    *   `src/app/core/concurrency.py`: Fallback logic for `multiprocessing` contexts (`spawn` vs `fork`) intended for legacy Linux environments or specific container runtimes. Simulating these OS constraints in CI creates flaky tests.

3.  **Production-Only Configuration:**
    *   `src/app/infrastructure/observability.py`: The initialization of the OTLP gRPC exporter. This code path only executes when a real OpenTelemetry collector is present (Production/Staging), which is bypassed in CI for speed.

## Decision
We adopt **"90%+ as the Gold Standard"**: the project must always run at 90% or higher line coverage. We explicitly reject writing fragile, mock-heavy tests solely to satisfy the coverage metric (Vanity Metrics) for the scenarios listed above.

## Consequences
*   **Positive:** Engineering effort is directed towards Observability (Metrics/Logs) and Feature development rather than maintaining brittle tests for theoretical edge cases.
*   **Negative:** There is a minute risk that a specific catastrophic driver failure logs a message incorrectly. This is deemed an acceptable operational risk, mitigated by our Integration Test suite running in CI.
*   **Enforcement:** The CI pipeline (`pyproject.toml [tool.coverage.report] fail_under = 90`) fails if coverage drops below 90%.

## Compliance
This decision satisfies SOC2 requirements for "Code Quality" by demonstrating a deliberate, risk-based approach to testing rather than blind metric chasing.
