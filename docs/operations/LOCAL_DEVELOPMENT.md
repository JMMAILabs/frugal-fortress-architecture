# Local Development & Onboarding Guide

Welcome to the Frugal Fortress. We enforce strict environment parity to eliminate "it works on my machine" issues. 

## 1. Prerequisites
*   **Docker & Docker Compose:** Required for the database, cache, and observability stack.
*   **uv (Astral):** We use `uv` for lightning-fast, deterministic Python dependency management.
*   **Python 3.12+:** The project is strictly pinned to 3.12 to ensure binary compatibility with NLP libraries.

## 2. Quickstart (Standard Setup)

1. **Clone and Configure:**
   ```bash
   git clone <repository-url>
   cd backend-monolith
   cp .env.example .env
   ```
   *Add your `GROQ_API_KEY` and `SECRET_KEY` to the `.env` file.*

2. **Spin up Infrastructure:**
   This starts PostgreSQL (pgvector), Redis, Loki, Tempo, and Grafana.
   ```bash
   docker compose up -d db redis tempo loki promtail grafana
   ```

3. **Install Dependencies & Setup DB:**
   ```bash
   uv sync
   uv run alembic upgrade head
   ```

4. **Run the Application:**
   You need two terminal windows.
   ```bash
   # Terminal 1: API Gateway
   uv run python -m uvicorn src.app.main:app --host 0.0.0.0 --port 8000 --reload

   # Terminal 2: Background Worker
   uv run arq src.app.worker.WorkerSettings
   ```

## 3. DevContainer (VS Code / GitHub Codespaces)
If you prefer a fully isolated environment, open the project in VS Code and click **"Reopen in Container"**. 
The `.devcontainer/devcontainer.json` will automatically install Python 3.12, `uv`, configure the Ruff linter, and attach to the Docker Compose network.

## 4. Running the Quality Gate
Before submitting a Pull Request, you must pass the Quality Gate.
```bash
# Formats code, runs Ruff, Mypy, and the Pytest suite
make dev

# Runs the full Mutation Testing suite (Warning: Takes ~5 minutes)
make mutation
```