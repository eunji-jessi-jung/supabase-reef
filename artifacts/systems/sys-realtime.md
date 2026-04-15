---
id: "SYS-REALTIME"
type: "system"
title: "Supabase Realtime Service"
domain: "realtime"
status: "draft"
last_verified: 2026-04-15
freshness_note: "Updated with dependency, runtime, and behavior details from application.ex, mix.exs, runtime.exs, router.ex, connect.ex, and authorization.ex"
freshness_triggers:
  - "config/runtime.exs"
  - "docker-compose.yml"
  - "lib/realtime/application.ex"
  - "lib/realtime/tenants/authorization.ex"
  - "lib/realtime/tenants/connect.ex"
  - "lib/realtime/tenants/replication_connection.ex"
  - "lib/realtime_web/channels/realtime_channel.ex"
  - "lib/realtime_web/router.ex"
  - "mix.exs"
known_unknowns:
  - "Exact message delivery guarantees and failure modes for Broadcast"
  - "How tenant rebalancing decides optimal node placement across regions"
  - "Performance characteristics of the Postgres CDC pipeline under load"
tags:
  - realtime
  - elixir
  - phoenix
  - websocket
  - cdc
aliases: []
relates_to:
  - type: "refines"
    target: "[[API-REALTIME]]"
  - type: "refines"
    target: "[[CON-AUTH-REALTIME]]"
  - type: "refines"
    target: "[[GLOSSARY-REALTIME]]"
  - type: "refines"
    target: "[[PROC-REALTIME-AUTH]]"
  - type: "refines"
    target: "[[RISK-REALTIME-KNOWN-GAPS]]"
  - type: "refines"
    target: "[[SCH-REALTIME]]"
  - type: "depends_on"
    target: "[[SYS-AUTH]]"
  - type: "feeds"
    target: "[[SYS-CLIENT-SDK]]"
sources:
  - category: "implementation"
    type: "github"
    ref: "config/runtime.exs"
  - category: "implementation"
    type: "github"
    ref: "docker-compose.yml"
  - category: "implementation"
    type: "github"
    ref: "lib/realtime/application.ex"
  - category: "implementation"
    type: "github"
    ref: "lib/realtime/tenants/authorization.ex"
  - category: "implementation"
    type: "github"
    ref: "lib/realtime/tenants/connect.ex"
  - category: "implementation"
    type: "github"
    ref: "lib/realtime/tenants/janitor.ex"
  - category: "implementation"
    type: "github"
    ref: "lib/realtime/tenants/replication_connection.ex"
  - category: "implementation"
    type: "github"
    ref: "lib/realtime_web/channels/realtime_channel.ex"
  - category: "implementation"
    type: "github"
    ref: "lib/realtime_web/router.ex"
  - category: "implementation"
    type: "github"
    ref: "mix.exs"
notes: ""
---

## Overview

Realtime is an Elixir/Phoenix server (v2.83.2) providing three real-time capabilities over WebSockets: Broadcast (ephemeral client-to-client messages), Presence (shared state tracking and synchronization), and Postgres Changes (CDC — Change Data Capture from Postgres WAL). It operates as a multi-tenant system where each Supabase project is a tenant with its own database connection, replication slot, and authorization policies. The service runs on Elixir 1.18 / OTP 27 and uses a forked Phoenix with custom presence dispatcher support.

## Key Facts

- Elixir 1.18 / OTP 27 application, version 2.83.2, using a Supabase-forked Phoenix with custom presence dispatcher → `mix.exs`
- Three features: Broadcast (ephemeral pub/sub), Presence (CRDT shared state), Postgres Changes (WAL CDC) — all GA → `README.md`
- Multi-tenant architecture: each tenant gets a dedicated `Connect` GenServer managing db_conn, replication_connection, and migration lifecycle → `lib/realtime/tenants/connect.ex`
- Cluster discovery uses `libcluster` with three strategies: DNS polling, Postgres-backed (prod default), and EPMD (dev) → `config/runtime.exs`
- Inter-node RPC uses `gen_rpc` (Supabase fork) with TCP or SSL transport; authorization checks transparently proxy to the node owning the db connection → `lib/realtime/tenants/authorization.ex`
- Process registry and tenant connection tracking uses `:syn` with scoped registries (`RegionNodes`, `Realtime.Tenants.Connect`) → `lib/realtime/application.ex`
- PubSub adapter is `Realtime.GenRpcPubSub`, routing messages across cluster nodes via gen_rpc rather than default Distributed Erlang → `lib/realtime/application.ex`
- Rate limiting enforced per-tenant: max bytes/sec (100K), max channels/client (100), max concurrent users (200), max events/sec (100), max joins/sec (100) → `config/runtime.exs`
- Authorization evaluates RLS policies by inserting probe rows into `realtime.messages` inside a transaction, then rolling back — separate read and write policy checks → `lib/realtime/tenants/authorization.ex`
- Replication connection uses `Postgrex.ReplicationConnection` with pgoutput plugin, protocol version 2, and automatic recovery with exponential backoff (max 2 hour window) → `lib/realtime/tenants/replication_connection.ex`, `lib/realtime/tenants/connect.ex`
- Janitor process runs scheduled cleanup of old messages per tenant, chunked and parallelized via `Task.Supervisor` → `lib/realtime/tenants/janitor.ex`
- WebSocket heap size capped at ~50MB (configurable via `WEBSOCKET_MAX_HEAP_SIZE`); full GC sweep forced every 20 reductions → `config/runtime.exs`, `config/config.exs`
- OpenTelemetry instrumentation for Cowboy, Phoenix, and Ecto with batch span processing → `lib/realtime/application.ex`, `config/config.exs`
- Tenant connections auto-shutdown after 6 consecutive zero-user checks (~5 minutes of inactivity) → `lib/realtime/tenants/connect.ex`
- Region-aware deployment: master region runs primary Repo; non-master regions use read replicas (FRA, IAD, SIN, SJC) → `config/runtime.exs`, `lib/realtime/application.ex`

## Responsibilities

- **Owns**: WebSocket connections, channel subscriptions, broadcast message routing, presence state synchronization, Postgres CDC subscription management, tenant lifecycle (connect, migrate, replicate, shutdown), rate limiting, RLS-based authorization for channels
- **Does not own**: User authentication (delegates to Auth JWTs), database schema management (delegates to tenant databases), object storage, admin dashboard UI beyond inspector/status pages

## Does NOT Own

| Concern | Owner | How Realtime Interacts |
|---------|-------|----------------------|
| User identity and JWT issuance | [[SYS-AUTH]] | Validates JWTs on WebSocket connect and channel join; extracts role/sub claims |
| Tenant database schema (application tables) | Tenant project | Reads WAL stream and runs RLS checks against tenant-owned tables |
| Client SDK packaging | [[SYS-CLIENT-SDK]] | realtime-js wraps the WebSocket protocol; Realtime defines the wire format |
| Platform metrics aggregation | External (Logflare, PromEx) | Pushes metrics via MetricsPusher; exposes /metrics endpoint |

## Dependencies

| System | Integration Type | Purpose | Auth Method |
|--------|-----------------|---------|-------------|
| PostgreSQL (primary) | Ecto + Postgrex | Tenant registry, `_realtime` schema storage | DB credentials (user/password) |
| PostgreSQL (per-tenant) | Postgrex.ReplicationConnection | WAL logical replication for CDC | DB credentials per tenant (encrypted with `DB_ENC_KEY`) |
| PostgreSQL (replicas) | Ecto Repo replicas | Read-only queries in non-master regions | DB credentials |
| Phoenix PubSub (gen_rpc) | In-process | Cross-node broadcast/presence message routing | Cluster membership |
| libcluster | Erlang distribution | Node discovery (DNS, Postgres, or EPMD) | Shared cluster config |
| Joken | Library | JWT decoding and claim validation | `API_JWT_SECRET`, `METRICS_JWT_SECRET` |
| OpenTelemetry | SDK | Distributed tracing (Cowboy, Phoenix, Ecto) | OTLP exporter config |
| Logflare | HTTP backend | Structured log shipping (optional) | `LOGFLARE_API_KEY` |
| Cachex | In-process | Tenant cache, rate counter, log throttle | N/A |
| NimbleZTA (Cloudflare) | HTTP | Dashboard ZTA authentication (optional) | `CF_TEAM_DOMAIN` |

## Domain Behavior Highlights

- **OTP supervision tree**: Single `one_for_one` supervisor with 20+ children including PartitionSupervisors for migrations, replication connections, and tenant connect processes — crash isolation per tenant → `lib/realtime/application.ex`
- **GenServer-per-tenant**: Each tenant connection is a `Connect` GenServer that sequences through `GetTenant -> CheckConnection -> ReconcileMigrations -> RegisterProcess` via a pipe chain, then starts replication — all in `handle_continue` callbacks for non-blocking init → `lib/realtime/tenants/connect.ex`
- **syn registry as distributed state**: Tenant connection metadata (db_conn pid, replication_conn pid, region) is stored in `:syn` rather than a central database, enabling O(1) lookups and cross-node visibility without extra RPCs → `lib/realtime/tenants/connect.ex`
- **RLS probe pattern**: Authorization checks INSERT a probe message row, run a SELECT filtered by RLS, then ROLLBACK — effectively using Postgres itself as the policy engine without persisting test data → `lib/realtime/tenants/authorization.ex`
- **Beacon for presence**: Custom `Beacon` library (local path dependency) handles sharded presence broadcasting with configurable shard count and broadcast intervals, decoupling presence fanout from the main PubSub → `lib/realtime/application.ex`

## Runtime Components

| Component | Tech Stack | Purpose | Entry Point |
|-----------|-----------|---------|-------------|
| Phoenix Endpoint | Phoenix 1.7 (forked) + Cowboy 2 | HTTP/WebSocket server on port 4000 | `RealtimeWeb.Endpoint` |
| Realtime Channel | Phoenix Channels | WebSocket topic handler for broadcast/presence/CDC | `lib/realtime_web/channels/realtime_channel.ex` |
| Tenant Connect | GenServer + PartitionSupervisor | Per-tenant DB connection lifecycle | `lib/realtime/tenants/connect.ex` |
| Replication Connection | Postgrex.ReplicationConnection | WAL streaming per tenant | `lib/realtime/tenants/replication_connection.ex` |
| Janitor | GenServer + Task.Supervisor | Periodic cleanup of old messages | `lib/realtime/tenants/janitor.ex` |
| Metrics Pusher | GenServer | Push metrics to external collector | `Realtime.MetricsPusher` |
| Metrics Cleaner | GenServer | Periodic cleanup of stale metrics | `Realtime.MetricsCleaner` |
| Cluster Supervisor | libcluster | Node discovery and membership | `Realtime.ClusterSupervisor` |
| PromEx | PromEx | Prometheus metrics collection and export | `Realtime.PromEx` |
| Presence | Phoenix.Presence | CRDT-based distributed presence tracking | `RealtimeWeb.Presence` |
| Beacon | Custom (local dep) | Sharded user-scope presence broadcasting | `beacon/` |
| Router | Phoenix Router | API and browser route dispatch | `lib/realtime_web/router.ex` |

## Core Concepts

- **Tenant**: A Supabase project. Each tenant has its own database connection config, extensions, and channel subscriptions.
- **Channel**: A named topic (prefixed `realtime:`) that clients subscribe to. Channels can have broadcast, presence, and postgres_changes features.
- **Broadcast**: Fire-and-forget message routing between clients subscribed to the same channel.
- **Presence**: CRDT-based shared state tracking — tracks who is connected and their state, with join/leave events.
- **Postgres Changes**: Listens to WAL logical replication and pushes INSERT/UPDATE/DELETE events to subscribed channels, filtered by RLS policies.
- **Extensions**: The `postgres_cdc_rls` extension processes WAL events through RLS filters before delivery.
- **Connect**: The per-tenant GenServer that manages the full lifecycle: DB connection, migrations, replication, and auto-shutdown on inactivity.

## Related

- [[SCH-REALTIME]] -- data model details
- [[API-REALTIME]] -- API and WebSocket reference
- [[PROC-REALTIME-AUTH]] -- authentication mechanism
- [[GLOSSARY-REALTIME]] -- domain terminology
- [[RISK-REALTIME-KNOWN-GAPS]] -- known risks
- [[CON-AUTH-REALTIME]] -- auth integration contract
- [[SYS-AUTH]] -- provides JWTs for WebSocket auth
- [[SYS-CLIENT-SDK]] -- realtime-js sub-package wraps Realtime
