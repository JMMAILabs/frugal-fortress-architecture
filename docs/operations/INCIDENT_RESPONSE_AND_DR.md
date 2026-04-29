# Incident Response & Disaster Recovery (DR)

This document outlines the Frugal Fortress strategy for maintaining high availability (99.9% SLA) and recovering from catastrophic infrastructure failures.

## 1. Recovery Objectives
*   **Recovery Time Objective (RTO):** < 5 minutes for stateless API nodes; < 30 minutes for database failover.
*   **Recovery Point Objective (RPO):** < 5 minutes for relational data (PostgreSQL WAL shipping); 0 minutes for ephemeral cache (Redis).

## 2. Component Failure Scenarios & Mitigations

### A. Redis Eviction or Crash
Redis is used for Caching, Rate Limiting, Idempotency, and the Arq Job Queue.
*   **Impact:** Background jobs may be delayed; rate limits fail open; idempotency checks miss.
*   **Mitigation:** 
    *   The API is designed to **Fail Open** for rate limits and caching if Redis is unreachable, ensuring synchronous HTTP requests still succeed.
    *   The `DLQRecoveryService` (Dead Letter Queue) ensures that if a worker crashes mid-job, the interaction log is safely parked in a DLQ stream for manual or automated replay (`make load-test` / Admin API).

### B. Primary LLM Provider Outage (e.g., Groq / Vertex AI goes down)
*   **Impact:** AI inference fails or times out.
*   **Mitigation:** 
    *   The `CircuitBreakerFactory` detects consecutive timeouts/5xx errors and transitions to the `OPEN` state.
    *   Traffic is instantly routed to the configured fallback model (e.g., from `gemini-2.5-pro` to `gemini-2.5-flash-lite`).
    *   If all cloud providers fail, the system returns a localized HTTP 503/206 gracefully, preventing cascading thread-pool exhaustion.

### C. PostgreSQL / Supabase Outage
*   **Impact:** Cannot read user profiles, verify PAYG balances, or save feedback logs.
*   **Mitigation:** 
    *   The `ResilientInteractionLogger` attempts to publish logs to Redis Streams first. If the DB is down, logs queue up in Redis.
    *   Read-heavy endpoints rely on Redis L1/L2 caching to serve stale data (Stale-While-Revalidate) until the DB connection is restored.

## 3. Incident Severity Levels
*   **SEV-1 (Critical):** API Gateway down, DB unreachable, or PII Scrubber failing (Fail-Secure triggers total block). *Action: Immediate page to on-call engineer.*
*   **SEV-2 (High):** Primary LLM down (running on fallbacks), Arq queue backing up. *Action: Investigate within 1 hour.*
*   **SEV-3 (Low):** Non-critical background tasks failing (e.g., PDF data lifecycle pruning). *Action: Next business day.* 