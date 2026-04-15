---
id: "CON-AUTH-REALTIME"
type: "contract"
title: "Auth to Realtime JWT Contract"
domain: "supabase-reef"
status: "draft"
last_verified: 2026-04-15
freshness_note: "snorkel-depth scan"
freshness_triggers:
  - "internal/tokens/"
  - "lib/realtime/tenants/authorization.ex"
known_unknowns:
  - "Exact JWT claims that Realtime relies on beyond role and sub"
  - "How Realtime handles token expiration for long-lived WebSocket connections"
  - "Whether Realtime disconnects clients on token revocation"
tags:
  - contract
  - jwt
  - websocket
aliases: []
relates_to:
  - type: "integrates_with"
    target: "[[SYS-AUTH]]"
  - type: "integrates_with"
    target: "[[SYS-REALTIME]]"
sources:
  - category: "implementation"
    type: "github"
    ref: "lib/realtime/tenants/authorization.ex"
notes: ""
---

## Parties

- **Producer**: Auth (issues JWTs with user identity and role claims)
- **Consumer**: Realtime (validates JWTs on WebSocket join and periodic re-auth)

## Key Facts

- Auth issues JWTs consumed by Realtime for WebSocket authentication → `internal/tokens/`
- Realtime validates JWT against per-tenant jwt_secret on channel join → `lib/realtime/tenants/authorization.ex`
- Token re-validated every 5 minutes for long-lived connections → `lib/realtime_web/channels/realtime_channel.ex`
- JWT claims used for RLS policy evaluation on broadcast and presence permissions → `lib/realtime/tenants/authorization/policies/`
- Each tenant stores its own jwt_secret in the control plane DB → `lib/realtime/api/tenant.ex`

## Agreement

Auth produces JWTs that Realtime validates using per-tenant secrets. The token must contain claims compatible with Realtime's authorization module. Realtime re-validates tokens periodically, unlike the single-shot validation that Storage and PostgREST perform per request.

## Current State

The contract is implicit. The 5-minute re-validation interval means token revocation takes up to 5 minutes to take effect on active WebSocket connections.

## Related

- [[SYS-AUTH]] -- JWT producer
- [[SYS-REALTIME]] -- JWT consumer
- [[PROC-AUTH-AUTH]] -- how Auth issues tokens
- [[PROC-REALTIME-AUTH]] -- how Realtime validates tokens
