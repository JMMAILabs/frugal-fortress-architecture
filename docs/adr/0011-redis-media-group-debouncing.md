# ADR 0011: Redis-backed Media Group Debouncing for Telegram Webhooks

**Date:** 2026-05-12
**Status:** Accepted

## Context
The Audio Notes module includes a native Support Ticketing system via Telegram (`/t` command). Users frequently send multiple screenshots at once (a "Media Group" in Telegram). 
Telegram's API does not batch these into a single webhook payload. Instead, it fires $N$ concurrent webhooks simultaneously, one for each photo. A naive implementation would process these independently, resulting in $N$ separate support tickets being created for a single user issue.

## Decision
We implemented a distributed debounce mechanism using Redis to aggregate concurrent webhooks into a single logical transaction.

1. **Pending State:** When a webhook with a `media_group_id` arrives, the backend attempts to create or append to a Redis JSON object (`ticket:tg_media_pending:audio_notes:{user_id}:{media_group_id}`).
2. **Debounce Loop:** The primary worker enters a non-blocking `asyncio.sleep` polling loop (max 1.5 seconds), waiting for the `attachment_count` in Redis to stabilize.
3. **Resolution:** Once stabilized (or if the max attachments limit of 10 is reached), the worker processes all collected `file_id`s, downloads the images, uploads them to S3/Local storage, and creates a *single* `ActiveTicket` in the database.

## Consequences
* **Positive:** Prevents race conditions and duplicate ticket creation. Provides a seamless, intuitive UX for the end-user.
* **Negative:** Introduces a deliberate 1.5-second latency to ticket creation when media groups are detected.
* **Resilience:** If Redis is down, the system falls back to processing the first image and dropping the rest, failing gracefully rather than crashing. 