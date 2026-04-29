# Receipt Parser — History & Decryption Flow

This flowchart demonstrates the data retrieval process for the `GET /api/v1/receipts/history` endpoint. It highlights the Application-Level Encryption (ALE) mechanism, showing how encrypted payloads stored in PostgreSQL are decrypted on-the-fly in the application memory before being returned to the client, ensuring Zero-Trust data security at rest.


```mermaid
flowchart TD
  A([Start]) --> B["GET /api/v1/receipts/history"]
  B --> C["Query params:\nuser_id (required)\nuser_role? = issuer|payer\nlimit (1..500)\noffset (>=0)"]
  C --> D["Router -> ReceiptRepository(session)"]
  D --> E["ReceiptRepository.list_by_user(user_id, user_role, limit, offset)"]
  E --> F["SQL: SELECT receipts\nWHERE receipts.user_id = user_id"]
  F --> G{"user_role filtered?"}
  G -->|yes| H["AND receipts.user_role = user_role"]
  G -->|no| I["No filter (both types)"]
  H --> J["ORDER BY receipts.created_at DESC"]
  I --> J
  J --> K["LIMIT/OFFSET pagination"]
  K --> L["Iterate ReceiptRecord\nrec.payload_encrypted (ALE)"]
  L --> M["_decrypt_payload(encrypted_b64)"]
  M --> N["ReceiptSchema already decrypted"]
  N --> O["Return array of receipts\n(model_dump -> JSON)"]
  O --> P([Frontend])
  %% UI Connection
  P -.-> Q["Tabs:\nIssued = user_role=issuer\nReceived = user_role=payer"]
```

