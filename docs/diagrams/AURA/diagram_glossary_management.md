```md
# Glossary Management — Pagination & Export

This state/sequence diagram illustrates how users manage their "Nested Learning Loop" rules directly from Telegram using inline keyboards.

```mermaid
flowchart TD
    Start([User types /rules]) --> Fetch[Fetch Page 1 from DB]
    Fetch --> Render[Render Inline Keyboard]
    
    Render --> Wait([Wait for Callback Query])
    
    Wait -->|Click 'Next Page'| Nav[Callback: nav:2]
    Nav --> CheckRL{Rate Limit Check<br/>(Redis)}
    CheckRL -->|Exceeded| 429[Show Alert: 'Too many requests']
    CheckRL -->|OK| Fetch2[Fetch Page 2 from DB]
    Fetch2 --> Render
    
    Wait -->|Click 'Toggle Rule'| Toggle[Callback: toggle:rule_id]
    Toggle --> DBUpdate[Update DB: is_active = false]
    DBUpdate --> CacheInv[Invalidate Redis Glossary Cache]
    CacheInv --> Fetch2
    
    Wait -->|Click 'Export'| Export[Callback: export_glossary]
    Export --> GenXLSX[Generate XLSX in Memory]
    GenXLSX --> SendDoc[Send Document via Telegram API]
    SendDoc --> Wait
```