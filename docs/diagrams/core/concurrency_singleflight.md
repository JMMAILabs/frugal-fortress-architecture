# Concurrency: Async SingleFlight Pattern
This sequence diagram illustrates how the `AsyncSingleFlight` pattern prevents "Cache Stampedes". When multiple concurrent requests ask the exact same question, only the first request (Owner) executes the heavy LLM/Retrieval logic. The others (Waiters) suspend execution and share the final result.

```mermaid
sequenceDiagram
    autonumber
    actor User1 as User A (Prompt X)
    actor User2 as User B (Prompt X)
    actor User3 as User C (Prompt X)
    participant Router as AdaptiveRouter
    participant SF as AsyncSingleFlight
    participant LLM as LLM / VectorDB

    User1->>Router: route_request("Prompt X")
    Router->>SF: do_with_owner(hash("Prompt X"))
    SF-->>Router: Lock Acquired (Is Owner = True)
    Router->>LLM: Execute RAG & Inference

    Note over User2, SF: 10ms later...
    User2->>Router: route_request("Prompt X")
    Router->>SF: do_with_owner(hash("Prompt X"))
    Note right of SF: Key exists. Task suspended.
    SF-->>Router: Wait for Owner Task

    Note over User3, SF: 20ms later...
    User3->>Router: route_request("Prompt X")
    Router->>SF: do_with_owner(hash("Prompt X"))
    SF-->>Router: Wait for Owner Task

    LLM-->>Router: LLMResponse (Cost: $0.02)
    Router->>SF: Task Completed

    Note over SF, Router: SingleFlight distributes result to all waiters
    SF-->>Router: Return to User 1 (Owner)
    SF-->>Router: Return to User 2 (Waiter)
    SF-->>Router: Return to User 3 (Waiter)

    Router-->>User1: Response (Cost: $0.02)
    Router-->>User2: Response (Cost: $0.00, model: cache-coalesced)
    Router-->>User3: Response (Cost: $0.00, model: cache-coalesced)
```