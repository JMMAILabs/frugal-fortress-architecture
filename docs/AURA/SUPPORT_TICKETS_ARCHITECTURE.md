# Support Tickets Architecture (Telegram Helpdesk)

## 1. Architectural Overview
The Audio Notes module includes a fully integrated, native Helpdesk system. Instead of forcing users to leave Telegram to send an email, they can open support tickets directly using the `/t` command. 

This system is built on strict **Hexagonal Architecture**, ensuring that the Telegram delivery mechanism is decoupled from the core ticketing logic and attachment storage.

## 2. The "Media Group" Debounce Problem
**The Challenge:** When a user sends multiple photos in Telegram (a "Media Group"), Telegram does not send a single webhook payload. Instead, it fires $N$ concurrent webhooks (one for each photo) almost simultaneously. A naive implementation would create $N$ separate support tickets.

**The Solution (Redis Debouncing):**
We implemented a distributed debounce mechanism using Redis.
1. When the first webhook arrives with a `media_group_id`, the backend creates a pending payload in Redis (`ticket:tg_media_pending:audio_notes:{user_id}:{media_group_id}`).
2. Subsequent webhooks for the same media group append their `file_id` to this Redis list.
3. The primary worker enters a non-blocking `while` loop (`asyncio.sleep`), polling Redis until the attachment count stabilizes or the TTL expires.
4. Once stabilized, the system processes all attachments as a single, cohesive Support Ticket.

## 3. Attachment Storage & Security
Attachments are not stored in the database. They are streamed directly from Telegram's servers and uploaded to an S3-compatible object storage (or local disk in dev) via the `SupportAttachmentStoragePort`.
* **Limits:** Max 10 attachments per message. Max 5MB per file. Max 25MB total per request.
* **Security:** Files are sanitized, and MIME types are strictly validated against an allowlist (`image/jpeg`, `image/png`, etc.) to prevent malicious uploads.

## 4. Admin REST API Integration
Support agents do not need access to Telegram. They interact with the system via the secure REST API (`/admin/tickets`).
* When an admin replies via `POST /admin/tickets/{ticket_id}/reply`, the `TelegramClient` adapter pushes the message back to the user's chat.
* The conversation history is maintained in Redis (`ticket:active:{ticket_id}`) for fast retrieval and context preservation.