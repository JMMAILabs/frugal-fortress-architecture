# ADR 0005: Async Test Scoping Strategy (Stability over Complexity)

**Date:** 2026-01-24
**Status:** Accepted
 
## Context
We needed to decide between test suite performance (Session Scope/Parallelism) and stability (Function Scope/Sequential) for our Asyncio + SQLAlchemy + Pytest stack.
  
## The Problem
Attempts to optimize test runtime encountered two critical blockers:
1.  **Session Scope:** Caused `RuntimeError: Future attached to a different loop` due to conflicts between `pytest-asyncio`, `asyncpg`, and `anyio` when sharing the Event Loop across tests.
2.  **Parallel Execution (`pytest-xdist`):** Caused `sqlalchemy.exc.IntegrityError` and `UndefinedTableError` because multiple workers attempted to modify the database schema concurrently on the same database instance.
 
## Decision
We enforce **Function Scope** for all async fixtures and run tests **Sequentially**.

```toml
# pyproject.toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
asyncio_default_fixture_loop_scope = "function"
```

 
Rationale (Empirical Evidence)

    Stability: 100% Pass rate. Each test gets a pristine, isolated environment.

    Performance: The suite runs in ~43 seconds (280 tests). This is highly acceptable for a CI pipeline.

    Simplicity: No need to manage complex database provisioning for parallel workers (e.g., creating test_db_1, test_db_2...).

Consequences

    Stability: 100%. Tests are fully isolated. No loop leakage.

    Performance: Sub-optimal. Database connections are established/torn down ~300 times per suite run.

    Mitigation: We use pytest-xdist (parallel execution) only for unit tests that do not touch the DB, if necessary in the future.

Future Roadmap: When to Retry Session Scope

Re-attempt the "Session Scope" optimization when ONE of the following criteria is met:

    Python 3.13+ Adoption: Python 3.13 introduces significant changes to the Global Interpreter Lock (GIL) and asyncio loop management which may resolve the underlying loop binding friction.

    pytest-asyncio v1.0+: The plugin maintainers are actively reworking the loop scoping logic. A major release may fix the loop_scope="session" flakiness.

    Migration to Psycopg3 (Async): Unlike asyncpg, psycopg (v3) is often more forgiving about Event Loop binding. If we migrate drivers, we should re-test session scoping.