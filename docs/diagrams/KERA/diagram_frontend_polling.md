# PDF to Anki — Concurrent Polling & Telemetry
This sequence diagram shows how the Next.js frontend handles multiple asynchronous tasks simultaneously without blocking the React render cycle, while tracking user attention for FinOps ROI metrics.

```mermaid
sequenceDiagram
    participant User
    participant UI as Next.js React
    participant Tracker as AttentionTracker
    participant API as FastAPI Backend
    
    User->>UI: Drop 3 PDFs
    UI->>API: POST /upload x3 concurrent
    API-->>UI: 202 Accepted with Task A, Task B, Task C
    
    par Poll Task A
        loop Every 3s
            UI->>API: GET /status/Task_A
            API-->>UI: processing
        end
    and Poll Task B
        loop Every 3s
            UI->>API: GET /status/Task_B
            API-->>UI: completed
            UI->>UI: Update DeckQueue state
        end
    and Poll Task C
        loop Every 3s
            UI->>API: GET /status/Task_C
            API-->>UI: pending
        end
    end
    
    loop Every 10s via Visibility API
        Tracker->>Tracker: Calculate active viewport time
        Tracker-->>API: sendBeacon /telemetry/attention
        Note right of API: Correlate LLM cost with active engagement time
    end
```
