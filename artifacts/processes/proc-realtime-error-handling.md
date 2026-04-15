---
id: "PROC-REALTIME-ERROR-HANDLING"
type: "process"
title: "Realtime Error Handling and Recovery Processes"
domain: "supabase-reef"
status: draft
last_verified: 2026-04-15
freshness_note: "Deep scan of realtime_channel.ex, user_socket.ex, connect.ex, broadcast_handler.ex, presence_handler.ex, and logging.ex"
freshness_triggers:
  - "lib/realtime/tenants/connect.ex"
  - "lib/realtime_web/channels/realtime_channel.ex"
  - "lib/realtime_web/channels/realtime_channel/broadcast_handler.ex"
  - "lib/realtime_web/channels/realtime_channel/logging.ex"
  - "lib/realtime_web/channels/realtime_channel/presence_handler.ex"
  - "lib/realtime_web/channels/socket_disconnect.ex"
  - "lib/realtime_web/channels/user_socket.ex"
known_unknowns:
  - "Whether clients receive structured error codes they can programmatically react to, or only human-readable strings"
  - "How error telemetry is aggregated and alerted on in production"
  - "Whether the connect_error_backoff_ms and channel_error_backoff_ms values are tunable per tenant or global only"
tags:
  - realtime
  - error-handling
  - recovery
  - websocket
aliases: []
relates_to:
  - type: "parent"
    target: "[[SYS-REALTIME]]"
  - type: "refines"
    target: "[[PROC-REALTIME-AUTH]]"
sources:
  - category: "implementation"
    type: "github"
    ref: "lib/realtime/tenants/connect.ex"
    notes: "Tenant connection lifecycle with replication recovery"
  - category: "implementation"
    type: "github"
    ref: "lib/realtime_web/channels/realtime_channel.ex"
    notes: "Channel join error handling, periodic re-auth, shutdown"
  - category: "implementation"
    type: "github"
    ref: "lib/realtime_web/channels/realtime_channel/broadcast_handler.ex"
    notes: "Broadcast-specific error paths"
  - category: "implementation"
    type: "github"
    ref: "lib/realtime_web/channels/realtime_channel/logging.ex"
    notes: "Structured logging with throttling and telemetry"
  - category: "implementation"
    type: "github"
    ref: "lib/realtime_web/channels/realtime_channel/presence_handler.ex"
    notes: "Presence error paths including rate limiting"
  - category: "implementation"
    type: "github"
    ref: "lib/realtime_web/channels/socket_disconnect.ex"
    notes: "Distributed socket disconnect across cluster"
  - category: "implementation"
    type: "github"
    ref: "lib/realtime_web/channels/user_socket.ex"
    notes: "Socket-level error handling including malformed messages"
notes: ""
---

## Purpose

Documents how Supabase Realtime handles errors across its WebSocket lifecycle: connection failures, channel join errors, runtime message errors, rate limit enforcement, and tenant database connection recovery. The system uses a layered approach with structured error codes, backoff-based retries, and graceful shutdown patterns.

## Key Facts

- Channel join has 20+ distinct error cases handled in a `with`/`else` block; each returns `{:error, %{reason: message}}` with a structured error code like `"InvalidJWTToken"`, `"ConnectionRateLimitReached"`, `"TenantNotFound"` → `lib/realtime_web/channels/realtime_channel.ex`
- Join errors are wrapped by `join_error/1` which adds a `Process.sleep(channel_error_backoff_ms())` delay before returning, mitigating brute-force or reconnect storms → `lib/realtime_web/channels/realtime_channel.ex`
- WebSocket connect errors in `UserSocket` also apply a configurable `connect_error_backoff_ms()` sleep before returning the error → `lib/realtime_web/channels/user_socket.ex`
- `UserSocket.handle_in/2` rescues `InvalidMessageError`, `Jason.DecodeError`, and generic exceptions from malformed WebSocket frames, logging them without crashing the socket process → `lib/realtime_web/channels/user_socket.ex`
- The entire channel join is wrapped in a `rescue` block that catches any unhandled exception, logs it as `"UnknownErrorOnChannel"`, and returns `join_error` rather than crashing → `lib/realtime_web/channels/realtime_channel.ex`
- Rate limit exceeded on events per second triggers `shutdown_response` which pushes a `"system"` event with status `"error"` to the client, logs a warning, and stops the channel with `:normal` reason → `lib/realtime_web/channels/realtime_channel.ex`
- Postgres CDC subscription failures use randomized backoff retry (1-3s default, 5-10s on error) via `DBConnection.Backoff`; malformed subscription params abort retry since retrying is pointless → `lib/realtime_web/channels/realtime_channel.ex`
- Replication connection recovery in `Connect` uses exponential backoff (`rand_exp`, 5s min, 5min max) with a 2-hour maximum recovery window; exceeding it terminates the connection entirely → `lib/realtime/tenants/connect.ex`
- Before reconnecting replication, `Connect` checks `pg_stat_activity` for existing `realtime_replication_connection` entries to avoid duplicate WAL senders → `lib/realtime/tenants/connect.ex`
- `SocketDisconnect.distributed_disconnect/1` uses `:erpc.multicall` across all cluster nodes with a 5-second timeout to disconnect all sockets for a tenant, sending `Process.exit(pid, :shutdown)` to each transport pid → `lib/realtime_web/channels/socket_disconnect.ex`
- Logging module provides throttled logging via `Cachex` with configurable `{max_count, window_ms}` per tenant+code, preventing log flooding from repeated errors → `lib/realtime_web/channels/realtime_channel/logging.ex`
- Error-level logs emit telemetry events at `[:realtime, :channel, :error]` with `code` and `tenant` metadata for metrics collection → `lib/realtime_web/channels/realtime_channel/logging.ex`
- Broadcast handler silently drops messages when write policy is `false` (no error sent to client); RLS policy errors are logged but do not crash the channel → `lib/realtime_web/channels/realtime_channel/broadcast_handler.ex`
- Presence has three distinct rate limits: per-tenant presence events/second, per-client presence events per window (default 5 calls / 30s), and payload size validation; exceeding any triggers channel shutdown → `lib/realtime_web/channels/realtime_channel/presence_handler.ex`
- Database connection process (`Connect`) monitors both db_conn and replication_conn via `Process.monitor`; if the db connection goes down, the entire Connect process terminates; if replication goes down, it enters recovery mode → `lib/realtime/tenants/connect.ex`

## Steps

1. **WebSocket connect error** -- `UserSocket.connect/3` returns `{:error, reason}`; the `connect_error/1` helper adds a backoff sleep before returning to slow reconnect attempts
2. **Channel join error** -- any failure in the `with` chain returns a structured error; `join_error/1` adds backoff sleep; the client receives `{:error, %{reason: message}}`
3. **Runtime rate limit exceeded** -- `handle_info(:update_rate_counter)` checks if `rate_counter.limit.triggered`; if true, calls `shutdown_response` which pushes a system error message and stops the channel
4. **Token expiry during re-auth** -- `:confirm_token` handler detects expired/invalid token and calls `shutdown_response`, terminating the channel gracefully
5. **Postgres CDC reconnect** -- `:postgres_subscribe` handler retries with backoff on failure; `nil` response triggers immediate retry; malformed params abort retry
6. **Replication recovery** -- `Connect` detects replication DOWN via monitor, enters recovery loop with exponential backoff, checks for zombie WAL senders, and retries up to 2 hours
7. **Tenant disconnect** -- `SocketDisconnect` broadcasts disconnect signal across all cluster nodes, killing all transport pids for that tenant

## Worked Examples

### Replication Connection Recovery

1. Replication connection process crashes; `Connect` receives `{:DOWN, ref, ...}` message
2. `update_syn_replication_conn(tenant_id, nil)` clears the replication pid from :syn metadata
3. `Backoff.backoff/1` calculates delay (starts at ~5s, grows exponentially up to 5min)
4. After delay, `:recover_replication_connection` fires
5. Checks `pg_stat_activity` for existing `realtime_replication_connection` -- if found, waits for old WAL sender to exit
6. If clean, starts new replication connection and resets backoff
7. If elapsed time exceeds 2 hours (`@max_replication_recovery_ms`), terminates the entire connection

### Rate-Limited Channel Shutdown

1. Client sends messages faster than `events_per_second` limit
2. Each message increments `GenCounter` via `count/1`
3. `:update_rate_counter` periodic check finds `rate_counter.limit.triggered == true`
4. `shutdown_response/2` pushes `{"event": "system", "payload": {"status": "error", "message": "Too many messages per second"}}`
5. Channel process stops with `:normal` reason via `{:stop, :normal, socket}`

## Related

- [[SYS-REALTIME]] -- parent system
- [[PROC-REALTIME-AUTH]] -- authentication errors are a subset of error handling
