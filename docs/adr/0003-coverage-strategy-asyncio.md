# ADR 0003: Coverage Strategy for Asyncio Artifacts

Date: 2026-01-19

## Status
**Superseded by [ADR-0004](0004-coverage-threshold-acceptance.md).** The original decision around acceptance of "phantom misses" in async tracing is preserved here for historical context, but the binding numerical threshold is now defined in ADR-0004.

## Context
Achieving 100% code coverage in Python 3.12+ with `asyncio` and `coverage.py` presents specific challenges due to "Eager Task Execution" and async generator exception handling.
Specifically, we observed "phantom misses" in:
1. `src/app/core/router.py`: Exception handling inside async generators (`yield`).
2. `src/app/core/prompts.py`: Immediate returns after `await` calls (Cache Hits).
3. `src/app/core/reranker.py`: List comprehensions inside conditional branches.

## Decision (Historical)
We will NOT write fragile "sleep-based" tests just to satisfy the tracer.
We use `# pragma: no cover` sparingly where integration tests have proven the logic works, but the line tracer fails to register the hit.

The numerical baseline is no longer set here. See [ADR-0004](0004-coverage-threshold-acceptance.md) for the canonical threshold (90%+).

## Consequences
+ Tests remain fast and deterministic.
+ Code remains clean of "test-only" logic.
- Coverage reports show a small gap (1-3 lines) that must be manually verified during code reviews.
