# Glossary Management — Pagination & Export

This state/sequence diagram illustrates how users manage their "Nested Learning Loop" rules directly from Telegram using inline keyboards.

```mermaid
%%{init: {'theme':'default', 'flowchart':{'nodeSpacing':40, 'rankSpacing':55}}}%%
flowchart TD
    Start([User types /rules]) --> Fetch[Fetch Page 1 from DB]
    Fetch --> Render[Render Inline Keyboard]

    Render --> Wait([Wait for Callback Query])

    Wait -->|Click Next Page| Nav["Callback: nav:2"]
    Nav --> CheckRL{"Rate Limit Check<br/>Redis"}
    CheckRL -->|Exceeded| Alert["Show Alert:<br/>Too many requests"]
    CheckRL -->|OK| Fetch2[Fetch Page 2 from DB]
    Fetch2 --> Render

    Wait -->|Click Toggle Rule| Toggle["Callback: toggle:rule_id"]
    Toggle --> DBUpdate["Update DB<br/>is_active = false"]
    DBUpdate --> CacheInv[Invalidate Redis Glossary Cache]
    CacheInv --> Fetch2

    Wait -->|Click Export| Export["Callback: export_glossary"]
    Export --> GenXLSX[Generate XLSX in Memory]
    GenXLSX --> SendDoc[Send Document via Telegram API]
    SendDoc --> Wait

    %% Styling (high-contrast, dark-mode friendly)
    classDef start fill:#c8e6c9,stroke:#1b5e20,stroke-width:2px,color:#0d2818
    classDef decision fill:#ffe0b2,stroke:#e65100,stroke-width:2px,color:#3e2723
    classDef action fill:#bbdefb,stroke:#0d47a1,stroke-width:1px,color:#0d1b4b
    classDef warn fill:#ffcdd2,stroke:#b71c1c,stroke-width:1px,color:#3e0a0a
    class Start,Wait start
    class CheckRL decision
    class Fetch,Fetch2,Render,Nav,Toggle,DBUpdate,CacheInv,Export,GenXLSX,SendDoc action
    class Alert warn
```

> **Why XLSX in memory?** The export must not touch disk (PII safety) and Telegram supports up to 50 MB documents.
> The CSV path is preferred for smaller exports; XLSX is used when users explicitly need spreadsheet
> formulas or formatted columns.
