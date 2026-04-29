# Future Coverage Fix Prompt

**Role:** Senior Python QA Engineer & Asyncio Specialist.

**Context:**
This project uses Python 3.12, FastAPI, and Asyncio. We have a 99% coverage suite.
There are persistent "phantom misses" in `coverage.py` reports related to:
1. **Async Generators:** Exception blocks inside `async def ... yield` are executed (proven by tests) but not marked as covered.
2. **Eager Execution:** Return statements immediately following an `await` in high-concurrency mocks are missed by the tracer.

**Target Files:**
*   `src/app/core/router.py`: Lines 195-196 (Stream exception handling).
*   `src/app/core/prompts.py`: Line 68 (Redis cache hit return).
*   `src/app/core/reranker.py`: Branch 116->123 (List comprehension optimization).

**The Task:**
When `coverage.py` releases better support for Python 3.12's `sys.monitoring` (PEP 669), refactor the test suite to remove `# pragma: no cover` markers.
1. Remove pragmas.
2. Update `pyproject.toml` coverage settings (potentially enabling `branch = true` with specific async plugins).
3. Verify 100% coverage without modifying the business logic.
