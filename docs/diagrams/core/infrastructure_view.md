# Infrastructure & Deployment View

This diagram depicts the containerized architecture as defined in `docker-compose.yml` (or Kubernetes manifests). It emphasizes the separation of concerns between the API, Worker, and Observability stack.

```mermaid
graph TD
    subgraph "Docker Host / K8s Node"

        subgraph "Application Layer"
            API[🐳 API Container<br/>Port: 8000]
            Worker[🐳 Worker Container<br/>Background Tasks]
        end

        subgraph "Data Persistence Layer"
            Redis[🐳 Redis<br/>Port: 6379]
            Postgres[🐳 Postgres 16<br/>Port: 5432]
            VolumeDB[💾 pg_data Volume]
        end

        subgraph "Observability Stack (Sidecars)"
            Promtail[🐳 Promtail<br/>Log Scraper]
            Loki[🐳 Loki<br/>Log Aggregation]
            Tempo[🐳 Tempo<br/>Distributed Tracing]
            Prometheus[🐳 Prometheus<br/>Metrics]
            Grafana[🐳 Grafana<br/>Visualization]
        end
    end

    %% Networking
    API -->|Read/Write| Redis
    API -->|SQL/Vector| Postgres
    Worker -->|Job Queue| Redis
    Worker -->|Insert Logs| Postgres
    Postgres --> VolumeDB

    %% Observability Flow
    API -.->|Metrics| Prometheus
    API -.->|Traces| Tempo
    Worker -.->|Traces| Tempo

    Promtail -->|Scrape Logs| API
    Promtail -->|Push| Loki

    Grafana -->|Query| Loki
    Grafana -->|Query| Tempo
    Grafana -->|Query| Prometheus

    %% Styling
    style API fill:#dcedc8,stroke:#33691e
    style Worker fill:#dcedc8,stroke:#33691e
    style Postgres fill:#bbdefb,stroke:#0d47a1
    style Redis fill:#ffcdd2,stroke:#b71c1c
```
