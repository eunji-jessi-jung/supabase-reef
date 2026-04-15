---
id: API-REALTIME
type: api
title: "Realtime API and WebSocket Interface"
domain: realtime
status: draft
last_verified: 2026-04-15
freshness_note: "Deep read of router.ex, realtime_channel.ex, broadcast_controller.ex, user_socket.ex, and OpenAPI spec. Covers REST admin API and WebSocket channel protocol."
freshness_triggers:
  - "lib/realtime_web/channels/realtime_channel.ex"
  - "lib/realtime_web/channels/user_socket.ex"
  - "lib/realtime_web/controllers/broadcast_controller.ex"
  - "lib/realtime_web/controllers/tenant_controller.ex"
  - "lib/realtime_web/router.ex"
known_unknowns:
  - "Exact Prometheus metric names exposed at /metrics endpoint"
  - "Full JWKS validation flow when jwt_jwks is set instead of jwt_secret"
  - "Dashboard authentication ZTA flow details (NimbleZTA.Cloudflare)"
tags:
  - elixir
  - realtime
  - rest-api
  - websocket
aliases: []
relates_to:
  - type: "parent"
    target: "[[SYS-REALTIME]]"
  - type: "depends_on"
    target: "[[SCH-REALTIME]]"
sources:
  - category: implementation
    type: github
    ref: "lib/realtime_web/channels/payloads/config.ex"
  - category: implementation
    type: github
    ref: "lib/realtime_web/channels/realtime_channel.ex"
  - category: implementation
    type: github
    ref: "lib/realtime_web/channels/user_socket.ex"
  - category: implementation
    type: github
    ref: "lib/realtime_web/controllers/broadcast_controller.ex"
  - category: implementation
    type: github
    ref: "lib/realtime_web/controllers/tenant_controller.ex"
  - category: implementation
    type: github
    ref: "lib/realtime_web/router.ex"
  - category: documentation
    type: doc
    ref: "sources/apis/realtime/openapi.json"
notes: ""
---

## Overview

Realtime exposes two interfaces: a REST API for tenant management and health checks, and a WebSocket interface (Phoenix Channels) for real-time messaging. The REST API has three access tiers: unauthenticated health checks, API-key-protected admin/tenant management, and tenant-scoped endpoints with per-tenant JWT auth plus RLS policy enforcement for private channels.

### API Profile

| Aspect | Value |
|--------|-------|
| Framework | Phoenix (Elixir) with OpenApiSpex |
| Transport | HTTP/1.1 REST + WebSocket (Phoenix Channels v2) |
| Auth | Bearer JWT -- three secrets: `api_jwt_secret` (admin), `metrics_jwt_secret` (metrics), tenant JWT (client) |
| Rate Limiting | Per-tenant: events/sec, joins/sec, max concurrent users, max channels per client, bytes/sec |
| Serialization | JSON (REST), JSON or binary (WebSocket via V2Serializer) |

## Key Facts

- Seven router pipelines partition auth: `browser`, `api` (admin JWT), `tenant_api` (AssignTenant + RateLimiter), `secure_tenant_api` (AuthTenant), `metrics` (metrics JWT), `open_cors` (CORS *), `openapi` (spec serving) → `lib/realtime_web/router.ex`
- Admin API `check_auth` plug validates Bearer token against `api_jwt_secret` and checks a configurable blocklist; returns 403 on failure → `lib/realtime_web/router.ex`
- WebSocket connect resolves tenant from hostname via `Database.get_external_id(host)`, validates JWT, checks suspension status, and enforces `TenantRateLimiters` before allowing the socket → `lib/realtime_web/channels/user_socket.ex`
- Channel topic pattern is `realtime:{sub_topic}` -- joining bare `realtime:` (no sub_topic) returns `TopicNameRequired` error → `lib/realtime_web/channels/realtime_channel.ex`
- Join payload has structured `config` with embedded schemas: `config.broadcast` (ack, self, replay), `config.presence` (enabled, key), `config.postgres_changes` (event, schema, table, filter), `config.private` (boolean) → `lib/realtime_web/channels/payloads/config.ex`
- Private channels enforce RLS policies via `Authorization.get_read_authorizations` on join and `get_write_authorizations` on broadcast; public channels skip policy checks → `lib/realtime_web/channels/realtime_channel.ex`
- Token refresh happens via `access_token` channel event -- on receive, policies are reset to nil, token re-validated, and postgres CDC re-subscribed with updated claims → `lib/realtime_web/channels/realtime_channel.ex`
- Automatic token expiry monitoring: `confirm_token` schedules re-validation every 5 minutes or at token expiry (whichever is sooner); expired tokens trigger channel shutdown → `lib/realtime_web/channels/realtime_channel.ex`
- REST broadcast endpoint (`POST /api/broadcast`) goes through `open_cors` + `tenant_api` + `secure_tenant_api` pipelines and delegates to `BatchBroadcast.broadcast/3` which validates payload size per tenant config → `lib/realtime_web/controllers/broadcast_controller.ex`
- Tenant CRUD uses upsert semantics: `create` action checks changeset validity then delegates to `update`; on first create, runs per-tenant Ecto migrations → `lib/realtime_web/controllers/tenant_controller.ex`
- `reload` action stops all CDC drivers, shuts down the Connect module, and disconnects all sockets for the tenant -- a hard reset without deleting config → `lib/realtime_web/controllers/tenant_controller.ex`
- Message replay on join: private channels can pass `config.broadcast.replay` with `since` (unix ms) and `limit` (max 25) to receive recent persisted broadcast messages after join completes → `lib/realtime_web/channels/realtime_channel.ex`

## Source of Truth

Route definitions in `lib/realtime_web/router.ex`. WebSocket channel logic in `lib/realtime_web/channels/realtime_channel.ex`. OpenAPI spec (extracted, tier 4) at `sources/apis/realtime/openapi.json`.

## Resource Map

### REST API

| Group | Method | Path | Handler | Auth | Notes |
|-------|--------|------|---------|------|-------|
| Health | GET | `/healthcheck` | PageController.healthcheck | None | |
| Metrics | GET | `/metrics` | MetricsController.index | metrics_jwt_secret | Prometheus format |
| Metrics | GET | `/metrics/:region` | MetricsController.region | metrics_jwt_secret | Regional subset |
| Tenants | GET | `/api/tenants` | TenantController.index | api_jwt_secret | |
| Tenants | POST | `/api/tenants` | TenantController.create | api_jwt_secret | Upsert semantics |
| Tenants | GET | `/api/tenants/:id` | TenantController.show | api_jwt_secret | |
| Tenants | PUT | `/api/tenants/:id` | TenantController.update | api_jwt_secret | |
| Tenants | DELETE | `/api/tenants/:id` | TenantController.delete | api_jwt_secret | Stops CDC + teardown |
| Tenants | POST | `/api/tenants/:id/reload` | TenantController.reload | api_jwt_secret | Hard reset |
| Tenants | POST | `/api/tenants/:id/shutdown` | TenantController.shutdown | api_jwt_secret | Stops Connect only |
| Tenants | GET | `/api/tenants/:id/health` | TenantController.health | api_jwt_secret | |
| Ping | GET | `/api/ping` | PingController.ping | Tenant JWT | |
| Broadcast | POST | `/api/broadcast` | BroadcastController.broadcast | Tenant JWT + RLS | Batch messages |
| OpenAPI | GET | `/api/openapi` | OpenApiSpex.RenderSpec | None | |
| Swagger | GET | `/swaggerui` | OpenApiSpex.SwaggerUI | None | |

### WebSocket Interface

| Feature | Channel Event | Direction | Description |
|---------|--------------|-----------|-------------|
| Join | `phx_join` | Client -> Server | Subscribe to `realtime:{topic}` with config params |
| Broadcast | `broadcast` | Bidirectional | Ephemeral message to all subscribers; private channels check write RLS |
| Presence | `presence` | Client -> Server | Track/untrack presence state |
| Presence Sync | `presence_state` | Server -> Client | Full presence state sent on join |
| Presence Diff | `presence_diff` | Server -> Client | Incremental presence changes |
| Postgres Changes | `postgres_changes` | Server -> Client | INSERT/UPDATE/DELETE events from WAL |
| Token Refresh | `access_token` | Client -> Server | Rotate JWT without reconnecting |
| System | `system` | Server -> Client | Status/error messages (subscribe confirm, shutdown, etc.) |

### Worked Example: REST Broadcast

**Request:**

```http
POST /api/broadcast HTTP/1.1
Host: myproject.supabase.co
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json

{
  "messages": [
    {
      "topic": "chat:lobby",
      "event": "new_message",
      "payload": {"user": "alice", "body": "hello"},
      "private": false
    }
  ]
}
```

**Response (success):**

```http
HTTP/1.1 202 Accepted
```

**Response (rate limit):**

```http
HTTP/1.1 429 Too Many Requests

{"error": "You have exceeded your rate limit"}
```

The broadcast controller validates each message via `BatchBroadcast`, groups by `private` flag, checks write RLS policies for private messages, and publishes via `TenantBroadcaster.pubsub_broadcast` to all connected WebSocket subscribers on matching topics.

### Worked Example: WebSocket Channel Join

```json
// Client sends phx_join to "realtime:chat:lobby"
{
  "topic": "realtime:chat:lobby",
  "event": "phx_join",
  "payload": {
    "access_token": "eyJ...",
    "config": {
      "broadcast": {"ack": true, "self": true},
      "presence": {"enabled": true, "key": "user-123"},
      "postgres_changes": [
        {"event": "INSERT", "schema": "public", "table": "messages"}
      ],
      "private": true
    }
  },
  "ref": "1"
}
```

```json
// Server responds with postgres_changes subscription IDs
{
  "topic": "realtime:chat:lobby",
  "event": "phx_reply",
  "payload": {
    "status": "ok",
    "response": {
      "postgres_changes": [
        {"id": 12345678, "event": "INSERT", "schema": "public", "table": "messages"}
      ]
    }
  },
  "ref": "1"
}
```

## Related

- [[SYS-REALTIME]] -- parent system
- [[SCH-REALTIME]] -- data model backing tenant config, messages, and subscriptions
