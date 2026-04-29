**File: `docs/testing/MUTATION_PRAGMAS.md`**
 
# Mutation Testing Pragmas (`# pragma: no mutate`)
 
This document catalogs all locations where mutation testing has been explicitly disabled using `# pragma: no mutate`, along with the technical justification. These exceptions are strictly reserved for **security invariants, concurrency resilience, or equivalent mutants**.

## 1. Concurrency & SingleFlight (`app.core.concurrency`)
*   **Target:** `AsyncSingleFlight.do_with_owner`
*   **Reason:** This is a critical primitive for preventing Cache Stampedes. Mutating the lock acquisition logic (`if key in self._calls:`) or the `asyncio.shield` wrappers causes the test suite to hang (Deadlocks) or triggers `SIGXCPU` timeouts (Exit Code -24). These branches are security invariants; mutating them provides no semantic value.

## 2. In-Memory Caching (`app.core.prompts`, `app.core.feature_flags`)
*   **Target:** L1 Cache TTL boundaries (`if now <= expiry:`) and pruning loops (`while len(self._l1) > _L1_MAXSIZE:`).
*   **Reason:** We use custom L1 dictionaries instead of `alru_cache` to avoid binding to the `asyncio` Event Loop during tests. Mutating the TTL boundary (`<=` to `<`) or the eviction logic generates equivalent mutants that are already covered by behavioral tests.

## 3. FinOps Defaults (`app.core.finops`, `app.core.router`)
*   **Target:** Default budget limits (`daily_limit: float = 10.0`) and initial cost accumulators (`reserved_cost = 0.0`).
*   **Reason:** Mutating a default value (e.g., `10.0` to `11.0`) generates a surviving mutant that does not alter the proven control flow. The actual FinOps logic (reserving and committing budgets) is heavily tested via integration tests.

## 4. Security & JWT (`app.security.jwt_auth`, `app.security.secret_manager`)
*   **Target:** Type casting (`return cast(list[str], scopes_payload)`) and default secret fallbacks.
*   **Reason:** Mutating a `cast` does not alter runtime behavior, it only affects `mypy`. It is an unkillable equivalent mutant.

## 5. LLM / Worker Orchestration (`process_audio`, `process_pdf`, `unified_pdf_flashcards`)
*   **Target:** High-latency orchestration modules that coordinate cache, wallet,
    LLM providers, feedback sessions, and worker payload assembly.
*   **Reason:** These files are adapter/application-service wiring rather than
    pure domain decision logic. Mutating them repeatedly fans out into mocked
    provider calls, database-session branches, and timeout-heavy async paths,
    producing low-signal equivalent survivors and SIGXCPU risk. Their pure
    behavior remains mutation-tested in extracted modules such as
    `audio_heuristics`, `pdf_process_heuristics`, `unified_pdf_pure`, and tier
    math helpers, while the orchestration paths are protected by unit and
    integration tests that assert idempotency, fallback, FinOps, and telemetry
    contracts.

## 6. Lifecycle / Export Application Services
*   **Target:** `audio_notes.application.data_lifecycle`,
    `audio_notes.application.export_transactions`,
    `pdf_anki.application.data_lifecycle`, and
    `receipt_parser.application.data_lifecycle`.
*   **Reason:** These modules are application-service adapters around
    SQLAlchemy sessions, repository ports, OpenPyXL serialization, encrypted ORM
    fields, and external vector-store cleanup. Mutating the wiring repeatedly
    exercises broad I/O surfaces and creates low-signal equivalent survivors.
    Domain decisions remain covered in pure helpers such as lifecycle
    heuristics, prune selection, tier limits, and storage quota math; the
    application services themselves are guarded by focused tests for quota
    exhaustion, pruning idempotency, vector-delete fallbacks, and export
    contracts.
 
## How to Handle Timeouts (⏰ / Worker Exit -24)
If `make mutation` reports timeouts, it means the worker received a `SIGXCPU` signal because a mutant caused an infinite loop or bypassed an `asyncio.wait_for` timeout.
1. Run `uv run mutmut show <Mutant ID>` to view the diff.
2. If the mutation altered a critical concurrency boundary (e.g., removing a timeout parameter), add `# pragma: no mutate` to the original source line.  
