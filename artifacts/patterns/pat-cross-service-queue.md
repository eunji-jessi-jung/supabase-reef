---
id: "PAT-CROSS-SERVICE-QUEUE"
type: "pattern"
title: "Cross-Service Queue and Async Job Pattern"
domain: "supabase-reef"
status: draft
last_verified: 2026-04-15
freshness_note: "scuba-depth cross-service pattern analysis"
freshness_triggers:
  - "src/internal/queue/queue.ts"
  - "src/internal/queue/event.ts"
  - "src/internal/queue/database.ts"
  - "src/storage/events/workers.ts"
  - "src/storage/events/lifecycle/webhook.ts"
  - "apps/studio/state/table-editor-operation-queue.types.ts"
  - "apps/studio/data/table-rows/operation-queue-save-mutation.ts"
known_unknowns:
  - "Whether Studio's operation queue has a maximum size or backpressure mechanism"
  - "How PgBoss dead-letter queue items are monitored or retried in production"
  - "Whether any other service besides Storage uses PgBoss"
tags:
  - pattern
  - cross-service
  - queue
  - async
  - pgboss
aliases:
  - "Job Queue Pattern"
relates_to:
  - type: "related"
    target: "[[SYS-STORAGE]]"
  - type: "related"
    target: "[[SYS-STUDIO]]"
  - type: "related"
    target: "[[PAT-CROSS-SERVICE-STORAGE]]"
sources:
  - category: "implementation"
    type: "github"
    ref: "src/internal/queue/queue.ts"
  - category: "implementation"
    type: "github"
    ref: "src/internal/queue/event.ts"
  - category: "implementation"
    type: "github"
    ref: "src/internal/queue/database.ts"
  - category: "implementation"
    type: "github"
    ref: "src/storage/events/workers.ts"
  - category: "implementation"
    type: "github"
    ref: "src/storage/events/lifecycle/webhook.ts"
  - category: "implementation"
    type: "github"
    ref: "apps/studio/state/table-editor-operation-queue.types.ts"
  - category: "implementation"
    type: "github"
    ref: "apps/studio/data/table-rows/operation-queue-save-mutation.ts"
notes: ""
---

## Overview

Two Supabase services implement queue/async-job patterns, but in fundamentally different ways. Storage uses a server-side persistent job queue backed by PgBoss (Postgres-based), while Studio uses a client-side in-memory operation queue for batching table editor mutations. The patterns serve different purposes: Storage's queue provides durable async processing with retries and dead-letter queues; Studio's queue provides optimistic UI updates with batched SQL execution.

## Key Facts

- Storage's `Queue` class wraps PgBoss with configurable retry (retryLimit: 20, retryBackoff: true), job expiration (23 hours), and maintenance interval (5 minutes) → `src/internal/queue/queue.ts`
- PgBoss operates in its own schema `pgboss_v10` with configurable connection pool (`pgQueueMaxConnections`) and statement timeout → `src/internal/queue/queue.ts`
- Storage registers 13 worker types covering webhooks, admin deletes, migrations, JWKS rotation, Iceberg catalog reconciliation, PgBoss upgrades, and catalog sync → `src/storage/events/workers.ts`
- Each Storage event type extends the `Event<T>` base class which provides `send()` (queue), `invoke()` (synchronous), and `invokeOrSend()` (try sync, fallback to queue) dispatch strategies → `src/internal/queue/event.ts`
- When the PgBoss queue is unavailable, Storage events fall back to synchronous execution if `allowSync` is true, providing fault tolerance at the cost of latency → `src/internal/queue/event.ts`
- Storage events can be disabled per-tenant via `disableEvents` config, checked in `shouldSend()` before enqueueing → `src/internal/queue/event.ts`
- Storage creates dead-letter queues for every normal queue with 30-day retention, using PgBoss's built-in dead-letter routing → `src/internal/queue/queue.ts`
- Queue polling uses a semaphore for concurrency control (`@shopify/semaphore`) with configurable concurrent task count per queue and configurable polling interval (default 5s) → `src/internal/queue/queue.ts`
- The `QueueDB` class wraps a `pg.Pool` with explicit transaction management (BEGIN/COMMIT/ROLLBACK) and error aggregation via `AggregateError` → `src/internal/queue/database.ts`
- Storage's Webhook worker sends lifecycle events to a configurable `WEBHOOK_URL` via axios with 4s timeout and keep-alive connection pooling → `src/storage/events/lifecycle/webhook.ts`
- Studio's operation queue is client-side with three operation types: `EDIT_CELL_CONTENT`, `ADD_ROW`, `DELETE_ROW`, each tracked with a UUID, tableId, and timestamp → `apps/studio/state/table-editor-operation-queue.types.ts`
- Studio batches queued operations into a single SQL transaction using `wrapWithTransaction()`, converting each operation to SQL and executing via `executeSql` → `apps/studio/data/table-rows/operation-queue-save-mutation.ts`
- Studio's queue has four states: `idle`, `pending`, `saving`, `error`, providing UI feedback on batch save progress → `apps/studio/state/table-editor-operation-queue.types.ts`

## Where It Appears

| Aspect | Storage (PgBoss) | Studio (Client-side) |
|--------|-----------------|---------------------|
| **Runtime** | Server-side, Postgres-backed | Client-side, in-memory |
| **Persistence** | Durable (survives restarts) | Ephemeral (lost on page reload) |
| **Retry** | Up to 20 retries with exponential backoff | No automatic retry |
| **Dead-letter** | Per-queue dead-letter with 30d retention | N/A |
| **Concurrency** | Semaphore-controlled per queue | Sequential batch save |
| **Dispatch** | Queue, sync, or hybrid (invokeOrSend) | Always batch then execute |
| **Use cases** | Webhooks, admin deletes, migrations, JWKS | Table cell edits, row add/delete |
| **Event structure** | Typed payload with tenant, version, region | Typed payload with table, identifiers |
| **Monitoring** | OpenTelemetry metrics (scheduled, completed, error) | Queue status state (idle/pending/saving/error) |

## Design Intent

Storage's PgBoss-based queue ensures durable, retriable async processing for operations that should not block the request path (webhooks, background deletes, migrations). The fallback-to-sync pattern prioritizes availability over strict async semantics. Studio's client-side queue optimizes the table editing UX by batching rapid cell edits into fewer round-trips, providing optimistic updates without server-side queue infrastructure.

## Trade-offs

- **Storage's Postgres queue**: Uses the same database for queue storage and data, creating coupling. PgBoss v10 migration overhead. Benefit: no additional infrastructure (Redis, RabbitMQ).
- **Sync fallback in Storage**: When queue is down, synchronous execution adds latency but preserves correctness. Events marked `allowSync: false` will throw instead.
- **Studio's ephemeral queue**: Operations can be lost on browser crash. No durability guarantee for unsaved edits. Trade-off is simplicity and fast UX.
- **No shared queue infrastructure**: Storage and Studio solve async processing independently with no shared abstraction, which is appropriate given their very different requirements.

## Agent Guidance

- When investigating Storage job failures, check the PgBoss dead-letter queue in the `pgboss_v10` schema. Failed jobs land there after exhausting all 20 retries.
- Storage event classes have a static `getQueueName()` method that returns the queue name used in PgBoss. Use this to query job status.
- The `invokeOrSend()` pattern in Storage means some events may execute synchronously if the queue is down. Check logs for `[Queue] Error invoking event synchronously, sending to queue` messages.
- Studio's operation queue is purely client-side. If a user reports lost edits, the queue may have been in `pending` state when the page was closed.

## Related

- [[SYS-STORAGE]] -- Storage system context
- [[SYS-STUDIO]] -- Studio system context
- [[PAT-CROSS-SERVICE-STORAGE]] -- Storage file handling pattern
