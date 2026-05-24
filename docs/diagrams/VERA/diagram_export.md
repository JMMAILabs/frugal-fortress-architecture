# VERA (Receipt Parser) — Export Flow (CSV & XLSX)

This flowchart demonstrates the data retrieval and export process for the `GET /api/v1/receipts/export` endpoint. It highlights how Application-Level Encryption (ALE) is decrypted on-the-fly in memory before streaming the file (CSV or Excel) to the client.

```mermaid
%%{init: {'theme':'default', 'flowchart':{'nodeSpacing':40, 'rankSpacing':60}}}%%
flowchart TD
    A([Start]) --> B["GET /api/v1/receipts/export<br/>Query: format csv or xlsx, user_role?"]
    B --> C["ReceiptRepository.list_by_user(...)"]
    C --> D["_decrypt_payload ALE<br/>for each ReceiptRecord"]
    D --> E{"format equals xlsx?"}

    E -->|Yes| F1["Build XLSX in memory openpyxl<br/>ws.append row"]
    E -->|No| F2["Build CSV in memory csv.writer<br/>writer.writerow row"]

    F1 --> G1["StreamingResponse<br/>media_type: application/vnd.openxmlformats..."]
    F2 --> G2["StreamingResponse<br/>media_type: text/csv"]

    G1 --> H([Download in frontend])
    G2 --> H

    %% Domain Context
    C -.-> I["Data decrypted on-the-fly<br/>via backend AES/ALE.<br/>Zero plaintext on disk."]

    %% Styling
    classDef start fill:#c8e6c9,stroke:#1b5e20,stroke-width:2px,color:#0d2818
    classDef decision fill:#ffe0b2,stroke:#e65100,stroke-width:2px,color:#3e2723
    classDef action fill:#bbdefb,stroke:#0d47a1,stroke-width:1px,color:#0d1b4b
    classDef note fill:#fff9c4,stroke:#f57f17,stroke-width:1px,color:#3e2723,stroke-dasharray: 5 5
    class A,H start
    class E decision
    class B,C,D,F1,F2,G1,G2 action
    class I note
```
