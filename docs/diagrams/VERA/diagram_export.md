# VERA (Receipt Parser) — Export Flow (CSV & XLSX)

This flowchart demonstrates the data retrieval and export process for the `GET /api/v1/receipts/export` endpoint. It highlights how Application-Level Encryption (ALE) is decrypted on-the-fly in memory before streaming the file (CSV or Excel) to the client.

```mermaid
flowchart TD
  A([Start]) --> B["GET /api/v1/receipts/export\nQuery: format (csv/xlsx), user_role?"]
  B --> C["ReceiptRepository.list_by_user(...)"]
  C --> D["_decrypt_payload(ALE)\nfor each ReceiptRecord"]
  D --> E{"format == 'xlsx'?"}
  
  E -->|Yes| F1["Build XLSX in memory (openpyxl)\nws.append(row)"]
  E -->|No (csv)| F2["Build CSV in memory (csv.writer)\nwriter.writerow(row)"]
  
  F1 --> G1["StreamingResponse\nmedia_type='application/vnd.openxmlformats...'"]
  F2 --> G2["StreamingResponse\nmedia_type='text/csv'"]
  
  G1 --> H([Download in frontend])
  G2 --> H
  
  %% Domain Context
  C -.-> I["Data decrypted on-the-fly\nvia backend AES/ALE.\nZero plaintext on disk."]