# System Verification & Observability Guide

This guide outlines the standard procedure to verify the integrity, performance, and observability of the AI Backend.

## 1. Prerequisites

Ensure the full stack is running.

### Option A: Full Docker (Recommended for Observability)
This mode ensures Loki (Logs) works correctly.
```bash
docker-compose up -d
```

### Option B: Hybrid Dev (Manual Process)
Useful for debugging code changes.
```bash
# Terminal 1 (API)
TESTING=false SECRET_KEY=dev-secret uvicorn src.app.main:app --host 0.0.0.0 --port 8000 --reload

# Terminal 2 (Worker)
TESTING=false SECRET_KEY=dev-secret arq src.app.worker.WorkerSettings
```

---

## 2. Smoke Tests (CLI)

### A. Health Check
Verify API and Database connectivity.
```bash
curl http://localhost:8000/api/v1/
# Expected: {"status":"ok", "database":"connected", ...}
```

### B. AI Inference (The "Smart" Path)
Forces the system to use the LLM (Groq/OpenAI).
```bash
curl -X POST "http://localhost:8000/api/v1/test-router" \
     -H "Content-Type: application/json" \
     -d '{"prompt": "Explain the theory of relativity to a 5 year old"}'
```
*   **Check:** Response should take >1s. `model_used` should be an LLM (not cache).

### C. Semantic Cache (The "Fast" Path)
Repeat the **exact same request** from step B.
*   **Check:** Response should take <10ms. `model_used` should be `cache-semantic` or `cache-exact`.

### D. Persistence Verification
Check if the background worker saved the interaction.

Create a script `check_db.py`:
```python
import asyncio, sys, os
sys.path.append(os.getcwd())
from sqlalchemy import select
from src.app.infrastructure.database import sessionmanager
from src.app.domain.feedback import FeedbackLog

async def check():
    sessionmanager.initialize()
    async with sessionmanager.session() as session:
        result = await session.execute(
            select(FeedbackLog).order_by(FeedbackLog.created_at.desc()).limit(5)
        )
        for log in result.scalars().all():
            print(f"[{log.created_at}] {log.model_used} | Cost: ${log.cost_usd} | Prompt: {log.user_prompt[:30]}...")

if __name__ == "__main__":
    asyncio.run(check())
```
Run it: `python check_db.py`

---

## 3. Observability (Grafana Stack)

Access Grafana at: [http://localhost:3000](http://localhost:3000) (admin/admin)

### A. Distributed Tracing (Tempo)
**Goal:** Visualize the request lifecycle and latency bottlenecks.
1.  Go to **Explore** -> Select **Tempo**.
2.  Query Type: **Search**.
3.  Filter by `http.method = POST`.
4.  Click on a trace ID.
5.  **Analyze:**
    *   Look for `router.route_request` span.
    *   Identify `litellm.completion` (External API call).
    *   Identify `db.execute` (Database latency).

### B. Metrics (Prometheus)
**Goal:** Monitor system throughput and error rates.
1.  Go to **Explore** -> Select **Prometheus**.
2.  Click **"Code"** (top right of query bar).
3.  Try these queries:
    *   **Throughput:** `rate(http_server_duration_milliseconds_count[1m])`
    *   **Latency:** `histogram_quantile(0.95, sum(rate(http_server_duration_milliseconds_bucket[5m]) by (le)))`
    *   **Errors:** `rate(http_server_duration_milliseconds_count{http_status_code=~"5.."}[5m])`

### C. Logs (Loki)
**Goal:** Deep dive into application logs.
*Note: Only works fully when running via `docker-compose`.*
1.  Go to **Explore** -> Select **Loki**.
2.  Select Label: `container_name` = `...api...` or `...worker...`.
3.  Filter for errors: `{container_name=~".+"} |= "error"`

---

## 4. Troubleshooting

| Symptom | Probable Cause | Fix |
| :--- | :--- | :--- |
| **404 on /metrics** | Endpoint not defined in `main.py`. | Add the Prometheus route manually. |
| **Loki is empty** | Running `uvicorn` manually. | Logs are in your terminal, not Docker. Run via `docker-compose up`. |
| **DB Check shows old data** | Worker hasn't run yet. | Wait 60s (cron) or restart worker to flush queue. |
| **ONNX Error in logs** | No GPU detected. | Normal behavior. System falls back to CPU. |

### Tempo: I only see flat lines (No Waterfall)
*   **Cause:** You are looking at `GET /metrics` or `GET /health` traces. These are too simple.
*   **Fix:** You need the Trace ID of a complex request (e.g., a POST to the router).
    1.  Check your terminal logs where `uvicorn` is running.
    2.  Find a log entry with `"path": "/api/v1/test-router"`.
    3.  Copy the `trace_id` from that JSON log.
    4.  Paste it into Tempo search bar.

### Prometheus: The query editor is confusing
*   **Cause:** Grafana defaults to "Builder" mode.
*   **Fix:** Click the **"Code"** button (top right of the query bar) to write raw PromQL.
