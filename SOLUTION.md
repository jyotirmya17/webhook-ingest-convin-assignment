# Webhook Ingestion Service - Solution

## What was broken, and why

Three defects were causing the symptoms observed in production:

1. **Duplicate call records & inflating call-counts (Concurrency / Idempotency):**
   The application used a Time-Of-Check to Time-Of-Use (TOCTOU) anti-pattern in `EventExists`. When identical webhooks (with the same `event_id`) arrived concurrently, multiple goroutines checked `EventExists`, got `false`, and proceeded to insert duplicate events and update `calls` and `account_stats` simultaneously.
   - *Fix:* I added a `UNIQUE` constraint on the `events (event_id)` column in Postgres (`migrations/002_unique_event_id.sql`). I then modified the `InsertEvent` query to use `ON CONFLICT (event_id) DO NOTHING` and removed the `EventExists` check entirely. By relying on Postgres to enforce uniqueness atomically, we eliminate the race condition.

2. **Recordings never marked processed (Context Cancellation):**
   The asynchronous task (`processRecording`) was launched using the HTTP request's `context.Context`. Since the handler returned `200 OK` almost instantly, the context was immediately canceled, causing `processRecording` to fail right away without executing its logic.
   - *Fix:* I replaced the context passed to the goroutine with `context.WithoutCancel(ctx)`, which decouples the background task's lifecycle from the incoming HTTP request. I also added proper error logging.

3. **In-flight tasks disappearing on deploy (Graceful Shutdown):**
   The main server implemented a graceful shutdown that only waited for the HTTP server (`srv.Shutdown`) to finish active requests, completely ignoring any background recording-processing goroutines spawned by the `Service`. When the process exited, these goroutines were forcefully killed.
   - *Fix:* I added a `sync.WaitGroup` to the `Service` struct to track in-flight background goroutines. I exposed a `svc.Shutdown(ctx)` method that blocks until the WaitGroup counter reaches zero, and called it in `main.go` during graceful termination.

## Why Postgres for deduplication instead of Redis

I chose **Postgres** with an `ON CONFLICT DO NOTHING` approach over Redis for the following reasons:
- **Transactional Integrity:** By using the primary data store (Postgres) as the source of truth for idempotency, we avoid dual-write inconsistencies. If we used Redis (`SETNX`) but Postgres failed to insert, we could end up permanently ignoring future redeliveries of an event that was never actually recorded.
- **Simplicity:** We are already using Postgres for persistence, and it is perfectly capable of ensuring uniqueness. By placing the `UNIQUE` constraint on `event_id`, the database guarantees data integrity at the lowest level, freeing the application layer from complex distributed locking mechanisms.
- **Race-Condition Elimination:** `INSERT ... ON CONFLICT DO NOTHING` natively handles concurrent writes without needing to manage locks, avoiding TOCTOU bugs completely.

## Scaling to 10,000 webhooks/second

If this service had to handle 10,000 requests per second, I would change the architecture to decouple ingestion from processing entirely:

1. **Ingest to a Message Broker:** Instead of writing directly to Postgres and triggering HTTP handlers asynchronously, the API would simply validate incoming webhooks and push them onto a fast streaming broker (like Kafka, Redpanda, or Redis Streams).
2. **Worker Pool:** A separate fleet of worker instances would consume from the message broker at a controlled rate to perform the heavy lifting (Postgres writes, stat aggregations, fetching recordings).
3. **Batch Inserts:** At 10k RPS, executing individual `INSERT`s is too slow. The workers would buffer messages and use Postgres bulk inserts (`COPY` or batched `INSERT`) to persist events and update aggregates efficiently.
4. **Idempotency Strategy:** Idempotency checking would shift to the worker level using Redis or a dedicated fast KV store for immediate duplication rejection, while still backing it up with a unique index in the persistent storage.
