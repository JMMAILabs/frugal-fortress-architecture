# Infrastructure & Deployment View

This diagram depicts the containerized architecture as defined in `docker-compose.yml` (or Kubernetes manifests). It emphasizes the separation of concerns between the API, Worker, and Observability stack.

```mermaid
%%{init: {'theme':'default', 'flowchart':{'nodeSpacing':50, 'rankSpacing':70}}}%%
graph TD
    subgraph Host ["Docker Host / K8s Node"]

        subgraph AppLayer ["Application Layer"]
            API["🐳 API Container<br/>Port: 8000"]
            Worker["🐳 Worker Container<br/>Background Tasks"]
        end

        subgraph DataLayer ["Data Persistence Layer"]
            Redis["🐳 Redis<br/>Port: 6379"]
            Postgres["🐳 Postgres 16<br/>Port: 5432"]
            VolumeDB["💾 pg_data Volume"]
        end

        subgraph Observability ["Observability Stack (Sidecars)"]
            Promtail["🐳 Promtail<br/>Log Scraper"]
            Loki["🐳 Loki<br/>Log Aggregation"]
            Tempo["🐳 Tempo<br/>Distributed Tracing"]
            Prometheus["🐳 Prometheus<br/>Metrics"]
            Grafana["🐳 Grafana<br/>Visualization"]
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

    %% Styling — explicit dark text for dark-mode readability
    classDef app fill:#dcedc8,stroke:#33691e,stroke-width:2px,color:#1b2e0a
    classDef store fill:#bbdefb,stroke:#0d47a1,stroke-width:2px,color:#0a1f4b
    classDef cache fill:#ffcdd2,stroke:#b71c1c,stroke-width:2px,color:#3e0a0a
    classDef volume fill:#fff9c4,stroke:#f57f17,stroke-width:2px,color:#3e2723
    classDef obs fill:#e1bee7,stroke:#4a148c,stroke-width:2px,color:#2a063e
    class API,Worker app
    class Postgres store
    class Redis cache
    class VolumeDB volume
    class Promtail,Loki,Tempo,Prometheus,Grafana obs
```
