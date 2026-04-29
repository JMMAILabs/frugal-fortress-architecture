# Database Schema (ERD)

This diagram illustrates the core data structures in PostgreSQL. Note the use of **Table Partitioning** on `feedback_log` to handle high-volume interaction logs efficiently over time.

```mermaid
erDiagram
    FEEDBACK_LOG ||--o{ FEEDBACK_LOG_PARTITION : "partitioned by date"

    FEEDBACK_LOG {
        uuid id PK
        uuid request_id UK "Indexed"
        string tenant_id "Multi-tenancy"
        string user_prompt
        string llm_response
        vector embedding "1536 dim (hnsw)"
        tsvector search_vector "GIN Index"
        float score "0.0 - 1.0"
        float cost_usd "FinOps"
        datetime created_at "Partition Key"
    }

    PROMPT_TEMPLATE {
        uuid id PK
        string name "Unique"
        string version
        text content "Jinja2 Template"
        bool is_active
    }

    %% Partitions (Conceptual)
    FEEDBACK_LOG_PARTITION {
        string partition_2024
        string partition_2025
        string partition_2026
    }

    %% Relationships
    FEEDBACK_LOG }|..|{ PROMPT_TEMPLATE : "implicitly linked by usage"

```
