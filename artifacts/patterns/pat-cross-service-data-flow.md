---
id: "PAT-CROSS-SERVICE-DATA-FLOW"
type: "pattern"
title: "Cross-Service Data Flow Pattern"
domain: "supabase-reef"
status: draft
last_verified: 2026-04-15
freshness_note: "scuba-depth cross-service pattern analysis"
freshness_triggers:
  - "packages/core/supabase-js/src/SupabaseClient.ts"
  - "packages/core/supabase-js/src/lib/fetch.ts"
  - "packages/core/auth-js/src/GoTrueClient.ts"
  - "packages/core/realtime-js/src/RealtimeClient.ts"
  - "src/http/plugins/jwt.ts"
  - "src/http/plugins/db.ts"
  - "src/internal/database/connection.ts"
  - "lib/realtime_web/channels/realtime_channel.ex"
  - "lib/realtime/tenants/authorization.ex"
  - "apps/studio/data/fetchers.ts"
  - "internal/tokens/service.go"
known_unknowns:
  - "How the API gateway (Kong/Envoy) routes requests to individual services"
  - "Whether Storage webhook events can trigger Realtime CDC subscriptions"
  - "Full event topology between services in production (edge functions, log drain, etc.)"
tags:
  - pattern
  - cross-service
  - data-flow
  - jwt-propagation
aliases:
  - "Inter-Service Communication Pattern"
relates_to:
  - type: "synthesizes"
    target: "[[PROC-AUTH-AUTH]]"
  - type: "synthesizes"
    target: "[[PROC-STORAGE-AUTH]]"
  - type: "synthesizes"
    target: "[[PROC-REALTIME-AUTH]]"
  - type: "synthesizes"
    target: "[[PROC-STUDIO-AUTH]]"
  - type: "synthesizes"
    target: "[[PROC-CLIENT-SDK-AUTH]]"
  - type: "related"
    target: "[[PAT-CROSS-SERVICE-AUTH]]"
  - type: "related"
    target: "[[PAT-CROSS-SERVICE-QUEUE]]"
  - type: "related"
    target: "[[PAT-CROSS-SERVICE-MIDDLEWARE]]"
sources:
  - category: "implementation"
    type: "github"
    ref: "packages/core/supabase-js/src/SupabaseClient.ts"
  - category: "implementation"
    type: "github"
    ref: "packages/core/supabase-js/src/lib/fetch.ts"
  - category: "implementation"
    type: "github"
    ref: "packages/core/auth-js/src/GoTrueClient.ts"
  - category: "implementation"
    type: "github"
    ref: "packages/core/realtime-js/src/RealtimeClient.ts"
  - category: "implementation"
    type: "github"
    ref: "src/http/plugins/db.ts"
  - category: "implementation"
    type: "github"
    ref: "src/internal/auth/jwt.ts"
  - category: "implementation"
    type: "github"
    ref: "lib/realtime_web/channels/realtime_channel.ex"
  - category: "implementation"
    type: "github"
    ref: "lib/realtime/tenants/authorization.ex"
  - category: "implementation"
    type: "github"
    ref: "apps/studio/data/fetchers.ts"
  - category: "implementation"
    type: "github"
    ref: "internal/tokens/service.go"
notes: ""
---

## Overview

Data flows through the Supabase ecosystem via three primary mechanisms: JWT propagation (Auth issues tokens consumed by all services), event emission (Storage lifecycle events, Realtime CDC), and API calls (Client SDK and Studio call service endpoints). The Client SDK acts as the orchestrator on the client side, managing the auth token lifecycle and injecting it into all sub-client requests. On the server side, each service independently verifies the JWT and translates it into Postgres session variables for RLS evaluation.

## Key Facts

- Auth issues JWTs with claims including `sub`, `role`, `aud`, `exp`, `aal`, `session_id`, `is_anonymous`, `app_metadata`, and `user_metadata`; these claims are the universal identity contract consumed by all services → `internal/tokens/service.go`
- Client SDK's `SupabaseClient` creates a `fetchWithAuth` wrapper that injects the access token from the auth session (or custom `accessToken` function) into all PostgREST, Storage, and Functions requests → `packages/core/supabase-js/src/SupabaseClient.ts`
- On `TOKEN_REFRESHED` and `SIGNED_IN` events, `SupabaseClient` calls `realtime.setAuth(token)` to propagate the fresh JWT to the WebSocket connection → `packages/core/supabase-js/src/SupabaseClient.ts`
- On `SIGNED_OUT` from a storage event (cross-tab), `SupabaseClient` calls `this.auth.signOut()` and resets Realtime auth, ensuring session termination propagates across browser tabs → `packages/core/supabase-js/src/SupabaseClient.ts`
- Realtime accepts mid-session token refresh via `access_token` event, which resets policies to `nil`, re-validates the JWT, re-evaluates RLS, and re-subscribes to Postgres CDC → `lib/realtime_web/channels/realtime_channel.ex`
- Storage translates JWT claims into Postgres session variables via `TenantConnection.setScope()`, setting 10 variables including `role`, `request.jwt.claims`, `request.jwt.claim.sub`, `request.method`, `request.path`, and `storage.operation` → `src/internal/database/connection.ts`
- Realtime translates JWT claims into Postgres session variables via `set_conn_config/2`, setting 6 variables including `role`, `realtime.topic`, `request.jwt.claims`, `request.jwt.claim.sub`, `request.jwt.claim.role`, and `request.headers` → `lib/realtime/tenants/authorization.ex`
- Storage and Realtime set overlapping but different Postgres session variables from the same JWT, meaning RLS policies must reference the correct variable name depending on which service executes the query → `src/internal/database/connection.ts` and `lib/realtime/tenants/authorization.ex`
- Studio injects auth tokens via `constructHeaders()` middleware on every API request, adding `Authorization: Bearer` and `X-Request-Id` headers → `apps/studio/data/fetchers.ts`
- Studio's API calls flow through the platform API proxy, not directly to service backends; the proxy handles routing to Auth, Storage, Realtime, and PostgREST → `apps/studio/data/fetchers.ts`
- Storage emits lifecycle events (ObjectCreated, ObjectRemoved, etc.) to PgBoss queues, which can trigger webhooks to external systems; these events include tenant ref, bucket ID, and object name → `src/storage/events/lifecycle/object-created.ts`
- Realtime's RLS policy evaluation tests authorization by inserting then rolling back test messages in the `realtime.messages` table, using the JWT claims injected via `set_conn_config` → `lib/realtime/tenants/authorization.ex`
- Client SDK initializes Realtime with `accessToken: this._getAccessToken.bind(this)`, creating a continuous data flow from Auth session to WebSocket auth → `packages/core/supabase-js/src/SupabaseClient.ts`
- When `accessToken` option is provided (server-side), `supabase.auth` is replaced with a Proxy that throws on access, preventing accidental client-side auth calls while still enabling token flow to sub-clients → `packages/core/supabase-js/src/SupabaseClient.ts`

## Where It Appears

### JWT Propagation Flow

```
Auth (GoTrue) --> issues JWT
  |
  +--> Client SDK stores in localStorage
  |     |
  |     +--> fetchWithAuth injects into PostgREST requests
  |     +--> fetchWithAuth injects into Storage requests
  |     +--> fetchWithAuth injects into Functions requests
  |     +--> realtime.setAuth(token) on TOKEN_REFRESHED
  |
  +--> Storage verifies JWT --> setScope() --> RLS in Postgres
  +--> Realtime verifies JWT --> set_conn_config() --> RLS in Postgres
  +--> Studio GoTrue session --> constructHeaders() --> Platform API
```

### Event Emission Flow

```
Storage object mutation
  |
  +--> PgBoss queue --> Webhook worker --> external webhook URL
  |
  +--> (Postgres trigger) --> Realtime CDC --> WebSocket subscribers
```

### Cross-Tab Sync Flow

```
Tab A: user signs out
  |
  +--> BroadcastChannel / storage event
  |
  +--> Tab B: receives SIGNED_OUT
       |
       +--> signOut() clears session
       +--> realtime.setAuth(null) clears WS auth
```

## Design Intent

The data flow architecture ensures that Auth is the single source of truth for identity, while each service independently enforces authorization via RLS. The JWT acts as a portable, self-contained identity credential that flows through the system without requiring cross-service calls. Event emission (PgBoss, CDC) enables loose coupling between services for async processing. Cross-tab sync via BroadcastChannel ensures consistent auth state across browser contexts.

## Trade-offs

- **JWT as the integration contract**: Simple and stateless, but changes to JWT claims require coordinating all services. Adding a new claim in Auth is not useful until Storage and Realtime start reading it.
- **Different Postgres session variables**: Storage and Realtime set different session variables from the same JWT, which means RLS policies may need to reference different variable names depending on the query origin. This creates an implicit coupling.
- **No direct service-to-service communication**: Services communicate indirectly through Postgres (RLS, CDC) and JWTs. This simplifies the architecture but makes it harder to implement cross-service transactions.
- **Client SDK as orchestrator**: The SDK is responsible for propagating token refreshes to Realtime. If the SDK fails to call `setAuth`, the WebSocket connection will use stale tokens.

## Agent Guidance

- When tracing a data flow issue, start at the Client SDK's `_getAccessToken()` method. This is the single source for access tokens used by all sub-clients.
- The JWT claims injected into Postgres by Storage (`request.jwt.claims`) and Realtime (`request.jwt.claims`) use the same variable name, but Storage also sets `storage.operation` and Realtime sets `realtime.topic`. RLS policies can use any of these.
- Cross-tab session sync only works when `persistSession: true` and `BroadcastChannel` is available. In server-side or React Native environments, cross-tab sync is not applicable.
- Storage lifecycle events (ObjectCreated, etc.) flow through PgBoss, not through Realtime CDC. They are separate event systems. Realtime CDC listens to `realtime.messages` table changes, not Storage events.
- When the `accessToken` option is used (server-side rendering), the auth module is replaced with a Proxy. Any attempt to call `supabase.auth.*` will throw, which is the intended behavior.

## Related

- [[PROC-AUTH-AUTH]] -- Auth token issuance
- [[PROC-STORAGE-AUTH]] -- Storage JWT-to-RLS translation
- [[PROC-REALTIME-AUTH]] -- Realtime JWT-to-RLS translation
- [[PROC-STUDIO-AUTH]] -- Studio auth middleware
- [[PROC-CLIENT-SDK-AUTH]] -- Client SDK token lifecycle
- [[PAT-CROSS-SERVICE-AUTH]] -- Authentication pattern that generates the JWTs flowing through the system
- [[PAT-CROSS-SERVICE-QUEUE]] -- Queue pattern used for Storage event emission
- [[PAT-CROSS-SERVICE-MIDDLEWARE]] -- Middleware that intercepts and augments the data flow
