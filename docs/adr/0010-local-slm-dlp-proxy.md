# ADR 0010: Data Loss Prevention (DLP) via Local SLM

**Date:** 2026-04-06
**Status:** Accepted

## Context
To comply with SOC2, GDPR, and HIPAA, we must ensure that Personally Identifiable Information (PII) and Protected Health Information (PHI) never leak to external LLM providers (e.g., Groq, OpenAI). 
While we use Microsoft Presidio with SpaCy for regex/NER-based scrubbing, complex contextual PII (e.g., "My boss, John, told me his password is...") requires semantic understanding.

## Decision
We implemented a **Local DLP Proxy** using a quantized Small Language Model (SLM) running directly on our backend infrastructure.
*   **Model:** `Llama-3.2-3B-Instruct-Q4_K_M.gguf` (4-bit quantization).
*   **Engine:** `llama-cpp-python` orchestrated via Microsoft Semantic Kernel.
*   **Execution:** Offloaded to a `CpuBoundExecutor` (ProcessPool) to prevent blocking the FastAPI `asyncio` event loop.

## Rationale
1. **Fail-Secure Mandate:** Relying on an external API (like OpenAI) to scrub data before sending it to *another* external API introduces unacceptable latency and a circular security dependency. If the local SLM fails, the system throws a `SecurityPolicyViolation` (HTTP 403) and blocks the request. It fails closed.
2. **FinOps:** Running a 3B parameter model locally on CPU costs $0 in API fees.
3. **Edge AI in the Backend:** The 4-bit quantized model requires <3GB of RAM, fitting comfortably within our Railway/Docker resource constraints.

## Consequences
*   **Positive:** Absolute guarantee of data residency for the scrubbing phase. High compliance posture.
*   **Negative:** CPU spikes during inference. Mitigated by strict concurrency limits in the `CpuBoundExecutor` and rate limiting.