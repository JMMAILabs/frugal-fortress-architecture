# Load Testing & Chaos Engineering

To guarantee our 99.9% SLA and validate our FinOps constraints, the Frugal Fortress employs continuous load testing and Chaos Engineering.

## 1. Load Testing with Locust
We use `Locust` to simulate high-concurrency traffic against the API Gateway.

**Execution:**
```bash
# Ensure the full Docker stack is running
make load-test
```
Access the Locust Web UI at `http://localhost:8089`.

**What constitutes a "Pass"?**
In our architecture, HTTP 429 (Too Many Requests) and HTTP 503 (Budget Exceeded) are **SUCCESS** metrics during a load test. They prove that our Redis Token Buckets and `BudgetManager` are actively protecting the system from Denial of Wallet (DoW) attacks. 
A test **fails** if we see HTTP 500 (Internal Server Error) or if the FastAPI Event Loop blocks (latency spikes > 10s for simple requests).

## 2. Validating AsyncSingleFlight (Cache Stampede Prevention)
To test our concurrency coalescing:
1. Configure Locust to send 500 concurrent requests with the *exact same* prompt and `tenant_id`.
2. **Expected Outcome:** 
   * The first request takes ~2 seconds (LLM Inference).
   * The other 499 requests hang for ~2 seconds, then return simultaneously.
   * Check Langfuse/Phoenix: You should see exactly **1 LLM call** made to the provider. The other 499 were served from memory via `AsyncSingleFlight`.

## 3. Chaos Engineering Scripts
Located in `scripts/`, these tools simulate catastrophic failures in staging environments.

*   **`chaos_rate_limit.py`:** Floods the `/test-router` endpoint with massive prompts to trigger the TPM (Tokens Per Minute) rate limiter. Validates that the system returns a structured 429 response instead of crashing the LLM provider.
*   **Simulating Provider Outage:** To test the Circuit Breaker, temporarily change the `GROQ_API_KEY` to an invalid string in `.env`. Send a request. The system should detect the 401/500 errors, open the circuit, and gracefully degrade to the fallback model (e.g., Vertex AI) without dropping the user's request.