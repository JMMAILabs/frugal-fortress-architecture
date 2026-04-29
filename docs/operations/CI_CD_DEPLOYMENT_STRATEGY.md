# CI/CD & Deployment Strategy

## 1. Overview
The "Frugal Fortress" utilizes a modern, automated deployment pipeline designed for high velocity, deterministic builds, and zero-downtime deployments. The architecture is split between Edge delivery (Vercel) and a containerized Cloud environment (Railway/K8s).

## 2. Continuous Integration (The "Zero-Compromise" Quality Gate)
Every Pull Request triggers a strict GitHub Actions pipeline. Code cannot be merged unless it passes our multi-layered Quality Gate, orchestrated via our `Makefile`:

1. **Static Analysis & Security:** 
   * `Ruff` enforces strict PEP-8 compliance.
   * `Bandit` scans the AST for hardcoded secrets and security vulnerabilities (`make security`).
   * `Mypy` runs in `--strict` mode. No `Any` types are allowed in the core domain.
2. **Unit & Integration Tests:** 
   * `Pytest` runs the full suite against an ephemeral PostgreSQL `pgvector` database (via `tmpfs` in Docker for instant I/O).
   * Coverage must remain above **90%**.
3. **Mutation Testing:** 
   * `Mutmut` actively modifies the AST during CI to ensure tests actually catch regressions, preventing "vanity coverage" metrics (`make mutation`).
4. **Contract Fuzzing:** 
   * `Schemathesis` reads our OpenAPI spec and fuzzes the endpoints to ensure the API strictly adheres to its declared contracts (`make contract`).
5. **LLM Evaluation (AI Quality):** 
   * `DeepEval` runs semantic checks on LLM outputs to ensure prompt modifications do not degrade extraction accuracy (`make eval`).

## 3. Continuous Deployment (CD)

### A. Backend Monolith (Docker)
*   **Containerization:** We use a multi-stage `Dockerfile` powered by Astral's `uv`. This ensures deterministic, lightning-fast dependency resolution.
*   **Image Optimization:** The final runtime image is stripped of build tools (like `gcc` and `g++`), reducing the attack surface and keeping the image size under 300MB.
*   **Zero-Downtime Migrations:** Database schema changes are managed via `Alembic`. Migrations are executed in a pre-deploy step (`make migrate`). We enforce additive-only schema changes (e.g., adding columns, not dropping them) to ensure the old version of the app continues to work while the new version rolls out.

### B. Frontend (Vercel / Edge)
*   **Strategy:** Push-to-deploy. Vercel automatically builds the Next.js application.
*   **Edge Caching:** Static assets and the GenUI project grid are cached at the Edge (CDN). API routes (`/api/v1/*`) are proxied to the backend monolith.