# Circuit Breaker State Machine
This state diagram shows the distributed Circuit Breaker logic. It highlights the "Fail Open" design: if the distributed cache (Redis) fails, the breaker falls back to local memory state to ensure the application doesn't crash.

```mermaid
stateDiagram-v2
    [*] --> CLOSED

    state CLOSED {
        [*] --> Distributed_Closed
        Distributed_Closed --> Local_Closed : Redis Timeout/Error
    }

    state OPEN {
        [*] --> Distributed_Open
        Distributed_Open --> Local_Open : Redis Timeout/Error
    }

    state HALF_OPEN {
        [*] --> Probe_Lock_Acquired
    }

    CLOSED --> OPEN : Failures >= Threshold (e.g., 5)
    OPEN --> HALF_OPEN : Recovery Timeout Expired (e.g., 30s)

    HALF_OPEN --> CLOSED : Probe Request Success
    HALF_OPEN --> OPEN : Probe Request Failed

    Note right of OPEN
        Fast-fails requests.
        Triggers Fallback Models
        (e.g., GPT-4o -> Llama-3)
    end note
```