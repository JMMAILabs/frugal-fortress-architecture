# 🏛️ The Frugal Fortress: Enterprise AI Architecture

![Architecture](https://img.shields.io/badge/Architecture-Modular_Monolith-8A2BE2)
![Pattern](https://img.shields.io/badge/Pattern-Strict_Hexagonal-blue)
![Compliance](https://img.shields.io/badge/Compliance-SOC2_%7C_GDPR-success)
![FinOps](https://img.shields.io/badge/FinOps-Optimized-FFaa00)

Welcome to the documentation repository for **The Frugal Fortress**, an enterprise-grade AI backend architecture designed to serve multiple AI products from a single, secure, and highly optimized Modular Monolith.

🌐 **[Read the Full Documentation (MkDocs) Here](https://tu-usuario.github.io/frugal-fortress-architecture/)**

---

## 🎬 Architecture Deep Dives (Video Demos)

Watch the technical showcases demonstrating the system's resilience, FinOps controls, and observability in real-time:

*   🎧 **[AURA: Audio Understanding & Retention Architecture](https://link-a-tu-video-aura.com)** - *The Corporate Memory Engine.*
*   📄 **[KERA: Knowledge Extraction & Retention Architecture](https://link-a-tu-video-kera.com)** - *The Async Knowledge Beast.*
*   🧾 **[VERA: Verified Expense & Receipt Architecture](https://link-a-tu-video-vera.com)** - *The Cost-Efficient Financial Extraction Engine.*

---

## 🛑 The Enterprise AI Trap

Scaling Generative AI applications typically results in two catastrophic outcomes:
1. **Skyrocketing Costs:** Naive LLM integrations destroy profit margins. Every unoptimized token call compounds at scale into runaway infrastructure spend.
2. **Compliance Nightmares:** Sending sensitive corporate data (PII/PHI) to third-party APIs violates SOC2, GDPR, and HIPAA, creating existential legal risk.

## 💡 The Solution: Frugal AI Philosophy

We engineered a **Modular Monolith** in Python (FastAPI) and Next.js, purpose-built for heavy document processing and financial data extraction. It operates under a strict "Frugal AI" philosophy, guaranteeing data privacy and high availability while maintaining a baseline infrastructure cost of just **$5/month**.

### Core Architectural Pillars

🛡️ **1. Enterprise Security (SOC2 Ready)**
*   **Zero Data Retention:** Paid tiers utilize Google Vertex AI under strict enterprise agreements—no training on your data, ever.
*   **Fail-Secure DLP Proxy:** A local, CPU-bound Small Language Model (Llama 3.2 3B) and Microsoft Presidio redact all PII before any external API call.
*   **Application-Level Encryption (ALE):** All sensitive data at rest is encrypted via AES-128-CBC (Fernet). Zero plaintext exposure to DBAs.

💰 **2. Predictable Economics (FinOps)**
*   **Edge Compute Offloading:** Image compression and basic validations are pushed to the browser (WASM/Canvas), reducing backend bandwidth and VLM token consumption by up to 95%.
*   **Multi-Tier Semantic Caching:** Redis (L1) and LanceDB (L2) serve repeated concepts in `<50ms`, bypassing the LLM entirely.
*   **Two-Phase Commit Wallet:** Atomic Lua scripts in Redis prevent Denial of Wallet (DoW) attacks for Pay-As-You-Go users.

⚙️ **3. Site Reliability Engineering (SRE)**
*   **Distributed Circuit Breakers:** Automated fallback to secondary models guarantees 99.9% uptime under degraded provider conditions.
*   **Async Burst Ingestion:** Heavy parsing is offloaded to isolated `Arq` process pools (`CpuBoundExecutor`), preventing database connection starvation during traffic spikes.

🧪 **4. Engineering & QA Maturity**
*   **Strict Hexagonal Architecture:** Business logic is 100% isolated from AI providers and delivery mechanisms.
*   **Zero-Compromise QA:** CI/CD pipeline with **Mutation Testing** (`mutmut`) targeting the Core Domain. Every critical path is empirically verified, not just covered.

---

## 📦 Bounded Contexts (The Modules)

This monolith consolidates distinct product domains, completely isolated from one another at the application layer:

### 1. AURA (Audio Notes)
Transforms unstructured voice messages into structured, actionable corporate memory via a direct Telegram Webhook integration. Features a **Nested Learning Loop** that learns user-specific jargon in real-time without fine-tuning.

### 2. KERA (PDF to Anki)
A heavy-compute, asynchronous pipeline designed to ingest massive corporate documents and generate high-signal, spaced-repetition flashcards. Features **DSPy Prompt Optimization** compiled offline to maximize study quality and minimize token usage.

### 3. VERA (Receipt Parser)
A hybrid AI architecture designed to extract structured financial data with maximum privacy and minimum cost. Leverages Edge computing for payload compression and Cloud orchestration with deterministic math validation and Human-in-the-Loop (HITL) self-correction.

---

## 🛠️ Running the Documentation Locally

This repository uses [MkDocs](https://www.mkdocs.org/) with the Material theme to generate the static documentation site.

```bash
# 1. Clone the repository
git clone https://github.com/JMMAILabs/frugal-fortress-architecture.git
cd frugal-fortress-architecture

# 2. Install MkDocs and the Material theme
pip install mkdocs-material

# 3. Serve the documentation locally
mkdocs serve


Then, open http://127.0.0.1:8000 in your browser to view the interactive documentation, including rendered Mermaid.js C4 models and sequence diagrams.

Designed and Architected by Javier Martinez - AI Solutions Architect.