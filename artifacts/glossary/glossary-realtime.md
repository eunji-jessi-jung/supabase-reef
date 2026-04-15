---
id: "GLOSSARY-REALTIME"
type: "glossary"
title: "Realtime Domain Glossary"
domain: "realtime"
status: "draft"
last_verified: 2026-04-15
freshness_note: "Deepened with code-level definitions from application.ex, connect.ex, gen_rpc modules, nodes.ex, authorization.ex, batch_broadcast.ex, message_dispatcher.ex, realtime_channel.ex, tenants.ex, users_counter.ex, replication_connection.ex"
freshness_triggers:
  - "lib/realtime/"
  - "lib/realtime_web/"
  - "config/runtime.exs"
  - "mix.exs"
known_unknowns:
  - "Some terms may have business meanings not captured in code"
  - "Beacon library internals (local path dependency, source not inspected)"
tags:
  - glossary
aliases: []
relates_to:
  - type: "parent"
    target: "[[SYS-REALTIME]]"
  - type: "refines"
    target: "[[GLOSSARY-SUPABASE-REEF]]"
  - type: "related"
    target: "[[DEC-REALTIME-GENRPC-PUBSUB]]"
  - type: "related"
    target: "[[SCH-REALTIME]]"
sources:
  - category: "implementation"
    type: "code"
    ref: "lib/realtime/application.ex"
  - category: "implementation"
    type: "code"
    ref: "lib/realtime/tenants/connect.ex"
  - category: "implementation"
    type: "code"
    ref: "lib/realtime/gen_rpc.ex"
  - category: "implementation"
    type: "code"
    ref: "lib/realtime/gen_rpc/pub_sub.ex"
  - category: "implementation"
    type: "code"
    ref: "lib/realtime/gen_counter/gen_counter.ex"
  - category: "implementation"
    type: "code"
    ref: "lib/realtime/nodes.ex"
  - category: "implementation"
    type: "code"
    ref: "lib/realtime/tenants/authorization.ex"
  - category: "implementation"
    type: "code"
    ref: "lib/realtime/tenants/replication_connection.ex"
  - category: "implementation"
    type: "code"
    ref: "lib/realtime/tenants/batch_broadcast.ex"
  - category: "implementation"
    type: "code"
    ref: "lib/realtime_web/channels/realtime_channel.ex"
  - category: "implementation"
    type: "code"
    ref: "lib/realtime_web/channels/realtime_channel/message_dispatcher.ex"
  - category: "implementation"
    type: "code"
    ref: "lib/realtime/tenants.ex"
  - category: "implementation"
    type: "code"
    ref: "lib/realtime/users_counter.ex"
  - category: "implementation"
    type: "code"
    ref: "lib/realtime/tenants/rebalancer.ex"
  - category: "implementation"
    type: "code"
    ref: "lib/realtime/tenants/connect/piper.ex"
  - category: "implementation"
    type: "code"
    ref: "config/runtime.exs"
notes: ""
---

## Terms

| Term | Definition | Code Location | See Also |
|------|-----------|---------------|----------|
| Tenant | A Supabase project instance. Each tenant has its own JWT secret, database connection, rate limits, and extension configuration. Identified by `external_id` (string), not the UUID primary key. | `lib/realtime/api/tenant.ex` | [[GLOSSARY-SUPABASE-REEF]], [[SCH-REALTIME]] |
| Channel | A named topic (prefixed `realtime:`) that clients subscribe to for broadcast, presence, or postgres changes. Public channels use topic `{tenant_id}:{sub_topic}`, private channels use `{tenant_id}-private:{sub_topic}`. | `lib/realtime_web/channels/realtime_channel.ex`, `lib/realtime/tenants.ex` | |
| Broadcast | Fire-and-forget ephemeral message routing between clients on the same channel. Messages are dispatched via PubSub and are not persisted unless the channel is private (in which case they go through `realtime.messages`). | `lib/realtime_web/channels/realtime_channel.ex`, `lib/realtime/tenants/batch_broadcast.ex` | |
| Presence | CRDT-based shared state tracking across connected clients. Provides join/leave/sync events. Uses a custom `Beacon` library for sharded user-scope broadcasting with configurable shard count and broadcast intervals. | `lib/realtime_web/channels/realtime_channel.ex`, `lib/realtime/application.ex` | |
| Postgres Changes | CDC (Change Data Capture) feature that streams INSERT/UPDATE/DELETE events from Postgres WAL to subscribed clients, filtered by RLS policies. Uses the `postgres_cdc_rls` extension driver. | `lib/realtime/postgres_cdc.ex`, `lib/realtime/tenants/replication_connection.ex` | |
| Extension | A feature module enabled per tenant. Currently `postgres_cdc_rls` for RLS-filtered Postgres changes. Configured via the `extensions` table with partially encrypted settings. | `lib/realtime/api/extensions.ex` | [[SCH-REALTIME]] |
| Connect | The per-tenant GenServer (`Realtime.Tenants.Connect`) that manages the full lifecycle: DB connection, migration reconciliation, replication startup, user count monitoring, and auto-shutdown on inactivity. Started under a `PartitionSupervisor` with `DynamicSupervisor` children. | `lib/realtime/tenants/connect.ex` | |
| Piper | A pipeline abstraction (`Realtime.Tenants.Connect.Piper`) that sequences Connect initialization steps: `GetTenant -> CheckConnection -> ReconcileMigrations -> RegisterProcess`. Each step implements the `Piper` callback and returns `{:ok, acc}` or `{:error, reason}`. Execution time per step is logged. | `lib/realtime/tenants/connect/piper.ex` | |
| Replication Connection | A per-tenant `Postgrex.ReplicationConnection` that reads the WAL stream for broadcast changes. Uses the `pgoutput` plugin with protocol version 2. Creates a TEMPORARY logical replication slot and a publication scoped to `realtime.messages`. | `lib/realtime/tenants/replication_connection.ex` | |
| GenRpcPubSub | Custom Phoenix.PubSub adapter that replaces Distributed Erlang for cross-node message transport. Implements two-tier region-aware fan-out: direct broadcast within region, single-hop to one node per remote region which re-fans locally. | `lib/realtime/gen_rpc/pub_sub.ex` | [[DEC-REALTIME-GENRPC-PUBSUB]] |
| gen_rpc | Supabase-forked RPC library providing TCP/SSL connections between BEAM nodes. Supports connection multiplexing (`MAX_GEN_RPC_CLIENTS`, default 5), key-based connection selection for ordering guarantees, and configurable timeouts. Used for all cross-node PubSub, authorization proxying, and remote function calls. | `lib/realtime/gen_rpc.ex`, `mix.exs`, `config/runtime.exs` | [[DEC-REALTIME-GENRPC-PUBSUB]] |
| Syn | Distributed process registry (`:syn` library) used for two scopes: `RegionNodes` (tracking which nodes are in which region) and `Realtime.Tenants.Connect` (tracking per-tenant connection metadata including `conn`, `replication_conn`, and `region`). Provides O(1) lookups and cross-node visibility. | `lib/realtime/application.ex`, `lib/realtime/tenants/connect.ex` | |
| GenCounter | ETS-backed atomic counter (`Realtime.GenCounter`) using `decentralized_counters: true` and `write_concurrency: :auto` for lock-free increment operations. Used for rate limiting (events/sec, joins/sec, bytes/sec, presence events/sec, authorization errors, connect errors). | `lib/realtime/gen_counter/gen_counter.ex` | |
| RateCounter | Higher-level rate-limiting abstraction built on top of GenCounter. Uses sliding-window buckets (configurable tick and bucket length) with `:avg` or `:sum` measurement modes. Triggers rate limits and fires telemetry events when thresholds are exceeded. | `lib/realtime/tenants.ex` | |
| Janitor | Background GenServer (`Realtime.Tenants.Janitor`) that periodically cleans up old messages per tenant. Uses `Task.Supervisor` for parallelized, chunked cleanup. Configurable via `JANITOR_SCHEDULE_TIMER_IN_MS` (default 4 hours). | `lib/realtime/tenants/janitor.ex`, `config/runtime.exs` | |
| Rebalancer | Module (`Realtime.Tenants.Rebalancer`) that checks whether a Connect process is running in the correct region. Compares current cluster membership against expected tenant region. Only triggers rebalancing when the cluster node set is stable (no recent changes). | `lib/realtime/tenants/rebalancer.ex` | |
| Beacon | Local path dependency (`./beacon`) providing sharded presence broadcasting. Used for user counting (`UsersCounter`) via `Beacon.join/3`, `Beacon.member_count/2`, and `Beacon.local_member?/3`. Configured with `users_scope_shards` (default 5) and `users_scope_broadcast_interval_in_ms` (default 10s). | `lib/realtime/users_counter.ex`, `lib/realtime/application.ex` | |
| UsersCounter | Module tracking connected clients per tenant across the cluster using Beacon's `:users` scope. Provides `tenant_users/1` (cluster-wide count), `local_tenant_counts/0` (node-local counts), and `already_counted?/2` (dedup check). Used for `max_concurrent_users` enforcement. | `lib/realtime/users_counter.ex` | |
| MessageDispatcher | Custom PubSub dispatch module (`RealtimeWeb.RealtimeChannel.MessageDispatcher`) that implements fastlane metadata for bypassing channel processes. Caches serialized messages per serializer within a dispatch cycle. Tracks replayed message IDs to avoid duplicate delivery. Increments presence counters on `presence_diff` events. | `lib/realtime_web/channels/realtime_channel/message_dispatcher.ex` | |
| Fastlane | Optimization where PubSub messages are sent directly to the WebSocket transport pid (bypassing the channel GenServer) using metadata tagged as `{:rc_fastlane, transport_pid, serializer, topic, log_level, tenant_id, replayed_message_ids}`. | `lib/realtime_web/channels/realtime_channel/message_dispatcher.ex`, `lib/realtime_web/channels/realtime_channel.ex` | |
| BatchBroadcast | Virtual Ecto schema (`embedded_schema`) for validating batched broadcast payloads. Embeds many messages, each with topic, event, payload, and private fields. Validates per-tenant payload size limits and rate limits before dispatching. Supports both public and private channels with RLS write checks for private messages. | `lib/realtime/tenants/batch_broadcast.ex` | [[SCH-REALTIME]] |
| Tenant Topic | Internal PubSub topic format: `{external_id}:{sub_topic}` for public channels, `{external_id}-private:{sub_topic}` for private channels. The `-private` suffix controls whether RLS authorization checks are enforced. | `lib/realtime/tenants.ex` | |
| Private Channel | A channel where `config.private = true` in the join payload. Private channels enforce RLS read/write policies via the Authorization module and support message replay. Public channels skip authorization entirely. | `lib/realtime_web/channels/realtime_channel.ex` | |
| RLS Probe | The authorization pattern where INSERT + SELECT + ROLLBACK is used to test Postgres RLS policies without persisting data. Read checks insert probe messages and SELECT with RLS filters; write checks attempt INSERT and check for `insufficient_privilege`. | `lib/realtime/tenants/authorization.ex` | [[SYS-REALTIME]] |
| PartitionSupervisor | OTP supervisor pattern used to partition dynamic supervisors by tenant ID hash. Three PartitionSupervisors exist: `Migrations.DynamicSupervisor`, `ReplicationConnection.DynamicSupervisor`, and `Connect.DynamicSupervisor`, each with `schedulers_online * 2` partitions by default. | `lib/realtime/application.ex`, `config/runtime.exs` | |
| Region | The deployment region of a Realtime node (e.g., `us-east-1`, `eu-west-2`). Nodes self-register in `:syn`'s `RegionNodes` scope. Tenant region is derived from extension settings and mapped to the closest Realtime region via `platform_region_translator/1`. | `lib/realtime/nodes.ex`, `config/runtime.exs` | |
| Master Region | The region running the primary Ecto Repo (`Realtime.Repo`). Non-master regions use read replicas. Controlled by `DB_MASTER_REGION` environment variable. If not set, defaults to the node's own region. | `config/runtime.exs`, `lib/realtime/application.ex` | |

## Disambiguation

| Ambiguous Term | Context A | Context B |
|----------------|-----------|-----------|
| Channel | Phoenix Channel (server-side process handling WebSocket topic) | Realtime Channel (the user-facing concept of a named topic with broadcast/presence/CDC features) |
| Extension | Ecto schema in `_realtime.extensions` table (tenant CDC config) | The runtime module that processes CDC events (e.g., `Extensions.PostgresCdcRls`) |
| Broadcast | The user-facing feature (fire-and-forget messaging) | `Phoenix.Socket.Broadcast` struct used internally for PubSub messages |
| Connection | DB connection (`Postgrex` pool member managed by Connect) | WebSocket connection (Cowboy transport process) |
| Registry | Elixir `Registry` (local, used for channel counting and replication connection lookup) | `:syn` registry (distributed, used for tenant Connect process metadata) |
| Region | AWS region where tenant database lives (e.g., `ap-southeast-1`) | Realtime deployment region where nodes run (mapped from AWS region via `platform_region_translator`) |

## Naming Conventions

- Elixir module naming follows `Realtime.*` (core) and `RealtimeWeb.*` (web layer) convention
- Tenant migrations use timestamp-prefixed filenames in `lib/realtime/tenants/repo/migrations/`
- Channel topics always prefixed with `realtime:` followed by user-defined topic name
- GenCounter keys use tuple format `{domain, metric, tenant_id}` e.g. `{:channel, :events, "abc123"}` → `lib/realtime/tenants.ex`
- Replication slot names follow `supabase_{schema}_{table}_replication_slot_{suffix}` pattern → `lib/realtime/tenants/replication_connection.ex`
- Publication names follow `supabase_{schema}_{table}_publication` pattern → `lib/realtime/tenants/replication_connection.ex`
- PubSub worker names are `:"#{pubsub}#{adapter_name}_#{number}"` → `lib/realtime/gen_rpc/pub_sub.ex`

## Related

- [[SYS-REALTIME]] -- parent system
- [[SCH-REALTIME]] -- data model details
- [[DEC-REALTIME-GENRPC-PUBSUB]] -- gen_rpc architecture decision
- [[GLOSSARY-SUPABASE-REEF]] -- unified cross-service glossary
