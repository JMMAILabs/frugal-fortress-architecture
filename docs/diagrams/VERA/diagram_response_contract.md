# Receipt Parser — Response Contract

This flowchart illustrates the decision tree for the `POST /api/v1/receipts/process` endpoint. It details how the system enforces tier limits, detects duplicate uploads via image hashing, and determines whether to return an `HTTP 200 OK` (Verified) or an `HTTP 206 Partial Content` (Requires Human Correction) based on the deterministic math validation.

 
```mermaid
flowchart TD
  A([Start]) --> B["POST /api/v1/receipts/process"]
  B --> C{"Load user + tier"}
  C --> D["ReceiptParserUserRepository.get_or_create_user(user_id)"]
  C --> E["receipt_repo.count_by_user(user_id)"]
  D --> F["_enforce_tier_limits(tier, receipt_count)"]
  E --> F
  F -->|"429 (daily limit)"| R429a["HTTP 429 Too Many Requests"]
  F -->|"429 (storage capacity)"| R429b["HTTP 429 Too Many Requests"]
  F -->|OK| G{"Dup detect (if image_hash is present)"}
  G -->|"yes, exists"| R409["HTTP 409 Conflict\n'This receipt was already processed'"]
  G -->|no| H["process_receipt_image(tier, image_base64)"]
  H --> I{"Validation result"}
  I -->|"receipt != None"| OK200["HTTP 200 OK\nstatus='verified'\nbody.receipt complete"]
  I -->|"best_effort != None"| OK206["HTTP 206 Partial Content\nstatus='requires_human_correction'\nbody.best_effort_receipt"]
  I -->|"neither receipt nor best_effort"| UNPROC200["HTTP 200 OK (decorator)\nstatus='unprocessable'\nbody.error_message"]
  %% Product clarifications
  OK206 --> J["Frontend: editable form\nand send correction (defined by UI)"]
  UNPROC200 --> K["Frontend: show error_message"]
```

