# Welcome to The Frugal Fortress

**The Frugal Fortress** is an enterprise-grade AI backend architecture designed to serve multiple AI products from a single, secure, and highly optimized Modular Monolith.

This documentation repository serves as the single source of truth for our engineering standards, architectural decisions, and operational playbooks.

---

## 🧭 Navigation Guide

Whether you are a CTO evaluating our security posture, or an SRE investigating our deployment strategy, use the guide below to navigate the documentation:

### 📊 Business & Strategy
*   **[Executive Summary](EXECUTIVE_SUMMARY.md):** High-level business value, ROI, and core architectural pillars.
*   **[FinOps & Billing](architecture/BILLING_AND_TIERS.md):** How we maintain a $5/month infrastructure baseline while scaling to thousands of users.

### 🏛️ Core Architecture
*   **[Hexagonal Decoupling](architecture/HEXAGONAL_DECOUPLING.md):** Our strict adherence to Ports & Adapters.
*   **[Security & SOC2](architecture/SECURITY_AND_COMPLIANCE_SOC2.md):** Application-Level Encryption (ALE) and Zero Data Retention policies.
*   **[Multi-Tenancy](architecture/MULTI_TENANCY_AND_ISOLATION.md):** How we guarantee B2B data isolation at the query level.

### 📦 Bounded Contexts (The Modules)
Deep dives into the specific business logic and flows for each independent product:
*   🎧 **[AURA (Audio Notes)](AURA/index.md):** The Corporate Memory Engine.
*   📄 **[KERA (PDF to Anki)](KERA/index.md):** The Async Knowledge Beast.
*   🧾 **[VERA (Receipt Parser)](VERA/index.md):** The Cost-Efficient Financial Extraction Engine.

### ⚙️ Operations & SRE
*   **[CI/CD Strategy](operations/CI_CD_DEPLOYMENT_STRATEGY.md):** Zero-downtime deployments and GitHub Actions.
*   **[Observability](operations/observability_and_logs.md):** Distributed tracing with OpenTelemetry, Tempo, and Loki.
*   **[Incident Response](operations/INCIDENT_RESPONSE_AND_DR.md):** Circuit Breakers, Fallbacks, and Dead Letter Queues.

### 📜 Architecture Decision Records (ADRs)
We document the *why* behind our technical choices. Start with our [ADR Index](adr/0001-record-architecture-decisions.md) to understand our engineering culture.