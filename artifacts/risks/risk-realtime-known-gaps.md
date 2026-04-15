---
id: "RISK-REALTIME-KNOWN-GAPS"
type: "risk"
title: "Realtime Known Risks, Gaps, and Backlog Candidates"
domain: "supabase-reef"
status: draft
last_verified: 2026-04-15
freshness_note: "Deep scan covering TODO/FIXME analysis, delivery guarantees, cluster RPC patterns, and replication recovery"
freshness_triggers:
  - "lib/realtime/"
  - "lib/realtime/adapters/postgres/decoder.ex"
  - "lib/realtime/api/tenant.ex"
  - "lib/realtime/gen_rpc.ex"
  - "lib/realtime/tenants/connect.ex"
  - "lib/realtime_web/channels/"
known_unknowns:
  - "Whether gen_rpc TCP connection pool exhaustion has been observed in production"
  - "Whether the 2-hour replication recovery window is sufficient for all failure modes"
  - "Impact of ROLLBACK AND CHAIN in authorization checks on database connection pool under high concurrency"
tags:
  - risk-catalog
  - realtime
aliases: []
relates_to:
  - type: "parent"
    target: "[[SYS-REALTIME]]"
  - type: "refines"
    target: "[[PROC-REALTIME-ERROR-HANDLING]]"
sources:
  - category: "implementation"
    type: "github"
    ref: "README.md"
    notes: "Explicit no-delivery-guarantee statement"
  - category: "implementation"
    type: "github"
    ref: "lib/realtime/adapters/postgres/decoder.ex"
    notes: "TODOs in WAL decoder"
  - category: "implementation"
    type: "github"
    ref: "lib/realtime/api/tenant.ex"
    notes: "TODO for infra migration cleanup"
  - category: "implementation"
    type: "github"
    ref: "lib/realtime/gen_rpc.ex"
    notes: "Cross-node RPC with gen_rpc and erpc"
  - category: "implementation"
    type: "github"
    ref: "lib/realtime/rpc.ex"
    notes: "Alternative RPC module with erpc"
  - category: "implementation"
    type: "github"
    ref: "lib/realtime/tenants/connect.ex"
    notes: "Replication recovery and tenant connection lifecycle"
  - category: "implementation"
    type: "github"
    ref: "lib/realtime_web/channels/realtime_channel.ex"
    notes: "Channel error handling and CDC subscription retry"
  - category: "implementation"
    type: "github"
    ref: "lib/realtime_web/channels/socket_disconnect.ex"
    notes: "Distributed disconnect with erpc.multicall"
notes: ""
---

## Description

Catalogs risk signals, architectural gaps, and known limitations found in the Realtime codebase during deep scanning. This update expands from the initial snorkel scan (3 signals in 2 files) to cover delivery guarantees, cluster partition handling, RPC failure modes, WAL decoder edge cases, and replication recovery boundaries.

## Key Facts

- README explicitly states "The server does not guarantee that every message will be delivered to your clients" -- this is a documented design choice, not a bug, but consumers needing exactly-once semantics must implement client-side reconciliation → `README.md`
- Postgres WAL decoder has a TODO to verify `Origin` message decoding correctness: "Verify this is correct with real data from Postgres" at line 177, meaning this code path may not have been validated against production data → `lib/realtime/adapters/postgres/decoder.ex`
- WAL decoder has a second TODO for handling blank `pg_catalog` namespace: "Handle case where pg_catalog is blank, we should still return the schema as pg_catalog" -- missing handling could cause schema misidentification for system types → `lib/realtime/adapters/postgres/decoder.ex`
- Tenant model changeset has a TODO marked "remove after infra update" for backwards-compatible extension key handling (`"extensions"` string vs `:extensions` atom), indicating incomplete migration → `lib/realtime/api/tenant.ex`
- `GenRpc.call` returns `{:error, :rpc_error, :badnode}` when the target node is not in `Node.list()`, but there is no automatic retry or failover to another node -- a transient cluster partition could leave authorization checks failing until the node reconnects → `lib/realtime/gen_rpc.ex`
- `GenRpc.multicall` latency reporting is skewed: if Node A takes 500ms to respond, Nodes B and C report at least 500ms regardless of their actual response time, as acknowledged in a code comment → `lib/realtime/gen_rpc.ex`
- `SocketDisconnect.distributed_disconnect` uses `:erpc.multicall` with a 5-second timeout; if a node is unreachable, the disconnect for that node silently fails (result mapped to `:error` atom with no retry) → `lib/realtime_web/channels/socket_disconnect.ex`
- Replication recovery has a hard 2-hour cap (`@max_replication_recovery_ms`); if the underlying database issue persists longer, the tenant connection is terminated and must be re-established from scratch → `lib/realtime/tenants/connect.ex`
- Postgres CDC subscription retry uses random backoff (5-10s on error), but if the subscription params are malformed (`{:malformed_subscription_params, _}`), retry is aborted and the channel stays subscribed without CDC -- the user gets an error push but the channel remains open → `lib/realtime_web/channels/realtime_channel.ex`
- The `gen_rpc` TCP connection pool is bounded by `:max_gen_rpc_clients` config; under heavy cross-node traffic, consistent hashing via `phash2(key, max_clients)` could lead to hot-spot connections if tenant distribution is skewed → `lib/realtime/gen_rpc.ex`
- `connect_error_backoff_ms` and `channel_error_backoff_ms` are stored in `:persistent_term` and are global, not per-tenant; a tenant experiencing auth issues applies the same backoff as all other tenants → `lib/realtime_web/channels/user_socket.ex`

## Findings

| # | Location | Signal | Theme | Severity |
|---|----------|--------|-------|----------|
| 1 | `lib/realtime/adapters/postgres/decoder.ex:177` | TODO: unverified Origin message decoding | Data handling | medium |
| 2 | `lib/realtime/adapters/postgres/decoder.ex:191` | TODO: blank pg_catalog namespace handling | Data handling | medium |
| 3 | `lib/realtime/api/tenant.ex:49` | TODO: remove after infra update | Configuration debt | low |
| 4 | `README.md:54` | No message delivery guarantee by design | Delivery semantics | high |
| 5 | `lib/realtime/gen_rpc.ex` | No retry/failover on :badnode RPC error | Cluster resilience | medium |
| 6 | `lib/realtime_web/channels/socket_disconnect.ex` | Silent failure on unreachable node disconnect | Cluster resilience | medium |
| 7 | `lib/realtime/tenants/connect.ex` | 2-hour hard cap on replication recovery | Recovery limits | medium |
| 8 | `lib/realtime_web/channels/realtime_channel.ex` | CDC stays disconnected on malformed params | Feature degradation | low |
| 9 | `lib/realtime/gen_rpc.ex` | Multicall latency reporting skew | Observability | low |
| 10 | `lib/realtime/gen_rpc.ex` | Potential TCP connection hot-spotting | Scalability | low |

## Themes

### No Delivery Guarantee (architectural, high)

The README explicitly states messages may not be delivered. This is a fundamental design choice for a WebSocket pub/sub system, but downstream consumers building critical data pipelines on Realtime must implement compensating mechanisms (polling, change data capture, idempotent processing).

**Impact if unaddressed:** Clients assuming reliable delivery may silently lose events. The broadcast replay feature (private channels only) partially mitigates this but is opt-in and bounded.

### Cluster Partition Handling (3 signals, medium)

Three separate mechanisms -- RPC calls, distributed disconnect, and replication recovery -- have gaps in handling cluster partitions or unreachable nodes. `GenRpc.call` fails hard on `:badnode`, distributed disconnect silently drops unreachable nodes, and there is no explicit netsplit detection or recovery protocol.

**Impact if unaddressed:** During cluster topology changes or network partitions, some tenants may experience authorization failures (if their DB connection is on an unreachable node), orphaned sockets (if disconnect can't reach all nodes), or extended replication downtime.

### Data Handling Edge Cases (2 signals, medium)

The Postgres WAL decoder has two unverified code paths for Origin messages and pg_catalog namespace handling. These affect how WAL logical replication events are parsed and forwarded to clients.

**Impact if unaddressed:** Uncommon Postgres types or system catalog references could be decoded incorrectly, leading to malformed change events sent to subscribed clients.

### Configuration Debt (1 signal, low)

A TODO in the tenant model changeset for backwards-compatible extension key handling suggests an incomplete infrastructure migration. This is low severity as the code handles both cases, but adds maintenance burden.

**Impact if unaddressed:** Continued dual-path code increases cognitive load and risk of divergent behavior between old and new configurations.

## Related

- [[SYS-REALTIME]] -- parent system
- [[PROC-REALTIME-ERROR-HANDLING]] -- error handling processes that mitigate some of these risks
