# ADR 0016: Preventing Head-of-Line Blocking in the Arq Queue
**Date:** 2026-08-31
**Status:** Accepted

## Context
A PDF upload sat "processing" indefinitely. The worker had consumed nothing for over two hours: queue depth climbed from 25 to 57, `jobs_ongoing` stayed at 0, and even the cron jobs had stopped firing. The worker process was alive and polling — it recorded health on schedule — but started no work at all.

Redis held exactly one orphan:

```
arq:in-progress:cron:worker_health_check:1788158220123   ttl 21251s
```

Three independent settings combined into a total stall:

1. **Arq marks a running job** with `arq:in-progress:{job_id}`, expiring after `in_progress_timeout_s` — the longest registered job timeout plus 10s. `PDF_PROCESS_JOB_TIMEOUT_SECONDS` was 30,000, so every lock, including a 20-second health-check cron's, carried an **8h20m** TTL.
2. **The worker had never shut down cleanly** — 24 starts, 0 shutdowns — so a hard kill mid-job stranded that lock with its full TTL.
3. **`queue_read_limit` was 1**, so each poll read only the single oldest job id: the poisoned one. Arq skips any job whose lock still exists, the iteration ended there, and nothing behind it was ever examined.

Any one of the three is survivable. Together they wedge the queue for eight hours, cron jobs included, and every asynchronous job silently accumulates behind the blockage.

A fourth finding emerged while investigating: the FinOps worker-recycling hook, which calls `os._exit(0)` every N jobs to reset heap fragmentation, **had never once executed**. Arq hands each job a fresh `ctx` copy (`ctx = {**self.ctx, **job_ctx}`), so the counter written onto `ctx` reset to 1 on every job and never reached the threshold. No `worker_recycling_after_lifetime_jobs` line exists in any log.

## Decision
Four changes. None of them raises concurrency — `max_jobs` stays at 1.

1. **`queue_read_limit` 1 → 20.** The worker can now step over a blocked head and keep draining. The window only widens what is *inspected*; arq sizes its semaphore at `max_jobs + 1` precisely so the loop can peek one past the cap and return, so this cannot block on a running job.

2. **Startup reconciles orphaned locks.** This sits alongside `_cleanup_zombie_tasks`, which already declares in-flight work dead on restart: if no document survives a restart, no arq lock does either. Both assume a single consuming worker, so `WORKER_CLEAR_STALE_ARQ_LOCKS=false` disables it before scaling out — otherwise a second worker's live jobs would be unlocked and run twice.

3. **Recycling moved from `on_job_end` to `after_job_end`**, with its counter moved from `ctx` to module scope. Fixing the counter alone would have *introduced* the very leak diagnosed above: arq awaits `on_job_end` **before** `finish_job` deletes the lock, so exiting from there strands the lock of the very job that triggered the recycle. `after_job_end` runs after `finish_job`.

4. **Stop sleeping on Groq quota inside the job.** With `max_jobs` at 1, a free-tier PDF waiting for capacity held the only worker slot for up to eight hours, blocking premium PDFs and cron jobs the whole time. When every key is rate-limited and the run has generated nothing yet, the rotation raises `GroqFreeTierDeferRequested` and `process_pdf_task` requeues itself with `_defer_by`, freeing the slot immediately.

   The opt-in is deliberate and per-call: **once a chunk has landed, the backoff sleeps in place as before**, because requeueing would discard that chunk and spend its quota a second time. This targets the case that actually filled the queue — jobs that start, find the daily quota already gone, and can do nothing.

### Timeouts this allowed

| | Before | After |
|---|---|---|
| `PDF_PROCESS_JOB_TIMEOUT_SECONDS` | 30,000 (8h20m) | **3,600** |
| `GROQ_PDF2ANKI_BACKOFF_MAX_WALL_SECONDS` | 28,800 (8h) | **1,800** |
| Resulting in-progress lock TTL | 8h20m | **~1h** |

The 30,000 figure was not arbitrary — it was derived as "the Groq backoff wall plus twenty minutes of real work". Lowering it in isolation would have killed legitimate free-tier PDFs mid-backoff. Only once the waiting moved off the worker could it come down. Observed real PDF runs finish in roughly two minutes.

## Consequences
* **Positive:** A stranded lock is no longer fatal — the worker steps over it, and clears it on the next restart.
* **Positive:** Lock TTLs drop from over eight hours to about one, cutting the blast radius of any future orphan.
* **Positive:** A PDF waiting on quota no longer occupies the single worker slot.
* **Operational:** The worker now genuinely recycles every 50 jobs (`WORKER_MAX_JOBS_PER_LIFETIME`, `0` disables). This is the intended FinOps behaviour and should reduce steady-state RSS, at the cost of a cold start every 50 jobs.
* **Negative:** A PDF that requeues re-runs its pipeline from the top, including the LlamaParse call and the embeddings. Bounded by `GROQ_PDF2ANKI_MAX_DEFERRALS` (24), past which the document fails with the existing `GROQ_FREE_TIER_TPD_ALL_KEYS` message. Caching the parsed markdown by `file_hash` would remove this rework and is the natural next step.
* **Fixed in passing:** spilled-to-disk uploads are no longer unlinked at read time (a requeued run needs those bytes), so every give-up path now releases them explicitly.

## Lesson
The failure was not in any one setting but in their interaction, and the first diagnosis was wrong: the recycling hook looked like the obvious culprit until the logs showed it had never run. Two of the three contributing values were reasonable in isolation, and the third — a job timeout sized for an eight-hour backoff — silently set the lock TTL for *every* job type, including a cron that takes twenty seconds. When a framework derives one global value from the maximum across all registrations, the largest outlier becomes everyone's blast radius.

Diagnosed while verifying [ADR 0015](0015-groq-gpt-oss-migration.md); the two are unrelated in cause.
