# 2. Coverage Strategy for Asyncio Artifacts

Date: 2026-01-19

## Status
Accepted

## Context
Achieving 100% code coverage in Python 3.12+ with `asyncio` and `coverage.py` presents specific challenges due to "Eager Task Execution" and async generator exception handling.
Specifically, we observed "phantom misses" in:
1. `src/app/core/router.py`: Exception handling inside async generators (`yield`).
2. `src/app/core/prompts.py`: Immediate returns after `await` calls (Cache Hits).
3. `src/app/core/reranker.py`: List comprehensions inside conditional branches.

## Decision
We accept a **99% coverage baseline**.
We will NOT write fragile "sleep-based" tests just to satisfy the tracer.
We use `# pragma: no cover` sparingly where integration tests have proven the logic works, but the line tracer fails to register the hit.

## Consequences
+ Tests remain fast and deterministic.
+ Code remains clean of "test-only" logic.
- Coverage reports show a small gap (1-3 lines) that must be manually verified during code reviews.
