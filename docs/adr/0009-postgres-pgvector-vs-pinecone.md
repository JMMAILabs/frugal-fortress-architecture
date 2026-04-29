Vector Search Strategy: Postgres (pgvector) & LanceDB vs. Pinecone

**Date:** 2026-04-06
**Status:** Accepted

## Context
Our system requires vector similarity search for two distinct use cases:
1. **Nested Learning Loop (Feedback):** Retrieving past user corrections to inject as few-shot examples into LLM prompts.
2. **PDF Semantic Deduplication:** Checking if a specific chunk of a PDF has already been processed to save LLM tokens.

We evaluated managed SaaS solutions like Pinecone and Qdrant against self-hosted/embedded solutions like PostgreSQL (`pgvector`) and LanceDB.

## Decision
We explicitly rejected Pinecone and adopted a **Hybrid Local Vector Strategy**:
1. **`pgvector` (PostgreSQL):** Used for the **Nested Learning Loop** (`FeedbackLog`).
2. **LanceDB (S3/R2 backed):** Used for **PDF Chunk Deduplication** (`pdf_anki_chunks`).
 
## Rationale
1. **Data Gravity & ACID Compliance:** Feedback logs contain relational metadata (tenant_id, timestamps, status) tightly coupled with the embedding. Using `pgvector` allows us to perform hybrid searches (Vector + SQL filtering) in a single ACID transaction without network hops or eventual consistency issues.
2. **Cost Efficiency (FinOps):** Pinecone introduces a high baseline cost ($70+/month) and charges per read/write. `pgvector` utilizes our existing Supabase/Postgres instance at **zero additional infrastructure cost**. LanceDB operates directly on cheap object storage (S3/Cloudflare R2), making it infinitely scalable for ephemeral PDF chunks at pennies per GB.
3. **Security & VPC Isolation (SOC2):** By keeping vectors inside our Postgres database and S3 buckets, sensitive embeddings never leave our Virtual Private Cloud (VPC). This drastically simplifies our SOC2 compliance audits compared to sending data to a third-party vector SaaS.

## Consequences
*   **Positive:** Massive cost savings, simplified infrastructure, strict data privacy, and transactional guarantees.
*   **Negative:** `pgvector` HNSW index build times can impact write performance at scale (>10M rows). We mitigate this by partitioning the `feedback_log` table by month.