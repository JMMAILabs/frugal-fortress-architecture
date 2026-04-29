# Support Tickets — Media Group Debouncing & Admin Reply

This sequence diagram illustrates how the system handles concurrent Telegram webhooks for multiple image uploads (Albums), debounces them using Redis (ADR-0011), and allows Admins to reply via the REST API.

```mermaid
%%{init: {'theme': 'default', 'themeVariables': { 'noteBkgColor': '#fff5ad', 'noteTextColor': '#333333'}}}%%
sequenceDiagram
    autonumber
    participant User as Telegram User
    participant API as FastAPI (Webhook)
    participant Cache as Redis (Debounce Guard)
    participant Storage as S3 / Local Storage
    participant Admin as Admin (REST API)

    Note over User, Cache: Phase 1: Media Group Debouncing (ADR-0011)
    User->>API: Send 3 Photos (Media Group ID: 123)
    
    par Concurrent Webhooks
        API->>Cache: Webhook 1 (Photo A) -> Create Pending List
        API->>Cache: Webhook 2 (Photo B) -> Append to List
        API->>Cache: Webhook 3 (Photo C) -> Append to List
    end
    
    Note right of API: Primary worker enters asyncio.sleep()<br/>polling loop (max 1.5s) for stability.
    API->>Cache: Fetch stabilized list (3 Photos)
    
    Note over API, Storage: Phase 2: Secure Storage & Ticket Creation
    loop For each Photo
        API->>Storage: Upload sanitized bytes
        Storage-->>API: Return Public URL
    end
    
    API->>Cache: Create Active Ticket (TTL: 30 days)
    API-->>User: "Ticket #45 created successfully."
    
    Note over Admin, User: Phase 3: Admin Resolution
    Admin->>API: POST /admin/tickets/45/reply {message: "Fixed!"}
    API->>Cache: Append to Ticket History
    API->>User: Send Telegram Message: "Support Team: Fixed!"
    API-->>Admin: 200 OK
```