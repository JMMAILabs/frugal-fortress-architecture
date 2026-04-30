# Hexagonal Architecture — Layer Decoupling

This document describes how **layer boundaries** and **dependency direction** are enforced in the Backend Monolith to maximise decoupling and testability.

---

## 1. Dependency Rule (Who May Import Whom)

```
  Primary Adapters (Routers, Workers)
           │
           ▼
  Application (Use Cases)  ──depends on──►  Domain (Ports, Entities)
           │                                        ▲
           │                                        │
           ▼                                        │
  Secondary Adapters (DB, Redis, LLM)  ──implements──┘
```

- **Domain** must not import from **Infrastructure**, **Core**, or **Adapters**.
- **Application** (use cases) must only depend on **Domain** (ports, entities) and **same-module** application/domain code.
- **Primary adapters** (routers, workers) may depend on **Dependencies** (composition root) and **Domain**; they should **not** import infrastructure directly for runtime behaviour — only via `app.dependencies`.
- **Secondary adapters** (infrastructure) implement **Ports** defined in domain (or module domain) and may depend on Domain for entity/metadata registration only where necessary.

### Summary Table

| Layer            | May import from                    | Must NOT import from        | 
|------------------|------------------------------------|-----------------------------|
| Domain           | stdlib, pydantic, (see §2)         | app.infrastructure, app.core, app.adapters |
| Application      | Domain, same module                | app.infrastructure, app.adapters (direct)  |
| Primary Adapters | app.dependencies, Domain, FastAPI  | app.infrastructure (prefer via dependencies) |
| Core             | Domain, Core, (facades: infra)     | app.adapters                |
| Infrastructure   | Domain, Core (config), third-party | —                           |
 
---

## 2. Domain Layer — Pragmatic Persistence

The **domain** holds:

- **Ports (interfaces):** `app.domain.interfaces` (e.g. `LLMProvider`, `CachePort`, `DatabasePort`).
- **Shared base for ORM:** `app.domain.base.Base` (SQLAlchemy `DeclarativeBase`).
- **ORM entities:** Some entities (e.g. `FeedbackLog`, `PromptTemplate`, `DocumentMetadata`) are implemented as SQLAlchemy models in domain for convenience.

**Rule:** Domain must not import **infrastructure services** (Redis clients, LiteLLM, FastAPI, HTTP clients). Use of SQLAlchemy in domain is limited to:

- `DeclarativeBase` and column/relationship definitions.
- No `Session`, `Engine`, or connection logic in domain.

For **stricter** decoupling, entities could be pure Python (or Pydantic) and a separate **infrastructure/persistence** layer would map them to tables; the current choice favours a single place for schema and avoids duplication.

**Module domain** (e.g. `app.modules.pdf_anki.domain`): May depend on `app.domain.base` and define module-specific **ports** and **entities**. Ports use `Protocol` or `ABC` and reference only domain types.

---

## 3. Application Layer (Use Cases)

- Lives under `app.modules.<name>.application`.
- Orchestrates flows by calling **ports** (injected or passed as arguments).
- Must **not** import `app.infrastructure` or `app.adapters`.
- May import from `app.domain` (shared interfaces) and from the same module’s `domain` and `application`.

Example: `process_pdf_use_case` receives `PdfParserPort`, `VectorStorePort`, `DocumentRepositoryPort`, `EmbeddingPort`, `FlashcardGeneratorPort`; it does not reference LiteLLM, SQLAlchemy sessions, or Redis.

---

## 4. Primary Adapters — Routers and Workers

- **Routers:** `app.modules.<name>.router` and `app.adapters.*`.
- **Workers:** `app.worker` (Arq tasks).

**Best practice:** Routers and workers obtain **all** runtime behaviour (DB session, cache, queue, LLM) via **dependency injection** from `app.dependencies`. Prefer:

```python
from app.dependencies import get_db, get_cache_port, get_queue_manager
```

and avoid:

```python
from app.infrastructure.database import get_db
from app.infrastructure.queue import QueueManager
```

so that primary adapters depend only on the **composition root** (`dependencies.py`), not on concrete infrastructure. Type hints for concrete classes (e.g. `QueueManager`) may still reference infrastructure when no domain port exists for that concern.

---

## 5. Composition Root (`app.dependencies`)

- **Single place** where infrastructure is wired to ports.
- Imports **domain interfaces** and **infrastructure implementations**.
- Exposes `get_*` functions used by FastAPI `Depends(...)`.
- Routers and main app should use only these `get_*` functions (and domain types), not infrastructure modules directly for resolving dependencies.

---

## 6. Core vs Infrastructure

- **Core** (`app.core`): Application-level services (retrieval, router logic, FinOps, prompts, classifier). Must not import **Adapters**; may import **Domain** and **Core**. The test suite allows **facades** (e.g. `core/database.py`, `core/redis.py`) that re-export from infrastructure for backward compatibility.
- **Infrastructure** (`app.infrastructure`): Implements ports (DB, Redis, LLM, queue, search). May import Domain (for Base/entities) and Core (e.g. config). One exception: global infrastructure may import a **module’s domain entities** only when required to register tables (e.g. `Base.metadata`); prefer keeping table registration close to the module (e.g. in module’s infrastructure or a dedicated bootstrap step) when possible.
- **Security** (`app.security`): Cross-cutting privacy, PII, guardrail, and
  encryption primitives. It must not import infrastructure or adapters so
  domain-adjacent privacy TypeDecorators can remain storage-agnostic. Legacy
  infrastructure facades may re-export security helpers during migration.

---

## 7. Module Bounded Contexts

Each module under `app.modules.<name>` follows the same layering:

| Folder           | Role                         | May import from                                      |
|------------------|------------------------------|------------------------------------------------------|
| `domain/`        | Entities, ports, pure logic  | app.domain (base), stdlib, pydantic                  |
| `application/`   | Use cases                    | Same module domain, app.domain.interfaces (if shared)|
| `infrastructure/`| Adapters for this module     | Module domain, app.domain, app.infrastructure, core  |
| `router.py`      | Primary adapter (HTTP)       | app.dependencies, app.domain, module application/domain, FastAPI |

Cross-module application-to-application imports are avoided; shared behaviour lives in **app.core** or **app.domain**.

---

## 8. Verification

- **Tests:** `tests/unit/test_architecture_compliance.py` enforces that **Domain** and **Core** do not import from Infrastructure/Adapters (with allowed facades).
- **Linting:** Use `import-linter` or similar to enforce layer boundaries (see project config).
- **Code review:** Check that new use cases receive ports (injected or passed) and that new routers use `app.dependencies` for injectables.

---

## 9. References

- [SYSTEM_TOPOLOGY.md](SYSTEM_TOPOLOGY.md) — High-level vision and module list.
- [Hexagonal Module Template (diagram)](../diagrams/core/hexagonal_module_template.md) — Diagram of a single module.
