# Support Tickets Architecture (Web Helpdesk)

## 1. Architectural Overview
The PDF to Anki module integrates with the Frugal Fortress native Helpdesk system. Instead of relying on external SaaS tools (like Zendesk), users can open support tickets directly from the Next.js frontend.
This system is built on strict **Hexagonal Architecture**, sharing the core support domain with other modules while using a REST API adapter for the web frontend.

## 2. Attachment Storage & Security
Users can attach screenshots or error logs to their tickets to help debug PDF parsing issues.
* **Limits:** Max 10 attachments per message. Max 5MB per file. Max 25MB total per request.
* **Security:** Files are sanitized, and MIME types are strictly validated against an allowlist (`image/jpeg`, `image/png`, `application/pdf`, etc.) to prevent malicious uploads.
* **Storage:** Attachments are uploaded to an S3-compatible object storage (or local disk in dev) via the `SupportAttachmentStoragePort`.

## 3. Rate Limiting & Abuse Prevention
To prevent spam, the support endpoints are protected by the Redis Token Bucket middleware:
* **Creation:** Limited to 6 tickets per minute (with a burst of 3 per 10 seconds).
* **Replies:** Limited to 10 replies per minute.
* **Business Logic:** A user can only have a maximum of 3 *active* tickets at any given time.

## 4. Admin REST API Integration
Support agents interact with the system via the secure REST API (`/admin/tickets`).
* When an admin replies via `POST /admin/tickets/{ticket_id}/reply`, the conversation history is updated in Redis (`ticket:active:{ticket_id}`).
* The user sees the response in real-time upon refreshing their web dashboard.