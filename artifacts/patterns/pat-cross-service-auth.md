---
id: "PAT-CROSS-SERVICE-AUTH"
type: "pattern"
title: "Cross-Service Authentication Pattern"
domain: "supabase-reef"
status: draft
last_verified: 2026-04-15
freshness_note: "scuba-depth cross-service pattern analysis"
freshness_triggers:
  - "internal/api/auth.go"
  - "internal/api/middleware.go"
  - "src/http/plugins/jwt.ts"
  - "src/internal/auth/jwt.ts"
  - "lib/realtime_web/channels/auth/jwt_verification.ex"
  - "lib/realtime_web/channels/auth/channels_authorization.ex"
  - "apps/studio/data/fetchers.ts"
  - "apps/studio/lib/auth.tsx"
  - "packages/core/supabase-js/src/lib/fetch.ts"
  - "packages/core/auth-js/src/GoTrueClient.ts"
known_unknowns:
  - "Whether all services handle JWKS key rotation the same way"
  - "How Auth admin role names are synchronized across services"
  - "Whether Realtime's 5-minute token re-validation is configurable per tenant"
tags:
  - pattern
  - cross-service
  - auth
  - jwt
aliases:
  - "JWT Auth Pattern"
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
sources:
  - category: "context"
    type: "documentation"
    ref: "sources/context/decisions/architecture-overview.md"
    notes: "Supabase product principles: 'everything works in isolation', 'everything is integrated'"
  - category: "implementation"
    type: "github"
    ref: "internal/api/auth.go"
  - category: "implementation"
    type: "github"
    ref: "internal/api/middleware.go"
  - category: "implementation"
    type: "github"
    ref: "src/http/plugins/jwt.ts"
  - category: "implementation"
    type: "github"
    ref: "src/internal/auth/jwt.ts"
  - category: "implementation"
    type: "github"
    ref: "lib/realtime_web/channels/auth/jwt_verification.ex"
  - category: "implementation"
    type: "github"
    ref: "lib/realtime_web/channels/auth/channels_authorization.ex"
  - category: "implementation"
    type: "github"
    ref: "apps/studio/data/fetchers.ts"
  - category: "implementation"
    type: "github"
    ref: "packages/core/supabase-js/src/lib/fetch.ts"
  - category: "implementation"
    type: "github"
    ref: "packages/core/auth-js/src/GoTrueClient.ts"
notes: ""
---

## Overview

Every Supabase service validates JWTs issued by Auth (GoTrue), but each uses a different mechanism suited to its runtime and protocol. Auth is the token issuer and also a token consumer for its own admin endpoints. Storage, Realtime, Studio, and the Client SDK all act as relying parties that extract, verify, and use JWT claims to enforce access control. The unifying contract is the JWT structure (sub, role, exp, aal, session_id) -- all services agree on this shape even though verification mechanics differ.

## Key Facts

- Auth extracts Bearer tokens via regex `(?i)^bearer (\S+$)` and parses with `jwt.NewParser` supporting `kid`-based key lookup with HS256 fallback for backward compatibility → `internal/api/auth.go`
- Auth admin authorization checks JWT `role` claim against `config.JWT.AdminRoles` list; any match grants admin privileges → `internal/api/auth.go`
- Storage extracts Bearer token by stripping `Bearer ` prefix via regex `/^Bearer\s+/i` and verifies using the `jose` library with dynamic algorithm selection from JWKS → `src/http/plugins/jwt.ts`
- Storage supports an `allowInvalidJwt` route config that lets unauthenticated requests proceed with `role: 'anon'` and `isAuthenticated: false` → `src/http/plugins/jwt.ts`
- Storage caches verified JWT payloads in an LRU cache (max 65536 items, 50 MiB, TTL matched to token exp) keyed by SHA-256 of token+secret+JWKS fingerprint → `src/internal/auth/jwt.ts`
- Realtime validates JWTs at two layers: WebSocket connect (UserSocket) and channel join (RealtimeChannel), using Joken signers with support for HS/RS/ES/Ed algorithm families → `lib/realtime_web/channels/auth/jwt_verification.ex`
- Realtime requires both `role` and `exp` claims present; missing either returns `{:error, :missing_claims}` → `lib/realtime_web/channels/auth/channels_authorization.ex`
- Realtime re-validates tokens every 5 minutes or at token expiry (whichever is sooner); expired tokens terminate the channel → `lib/realtime_web/channels/realtime_channel.ex`
- Studio injects auth via openapi-fetch middleware: `constructHeaders()` calls `getAccessToken()` and sets `Authorization: Bearer` and `X-Request-Id` headers on every request → `apps/studio/data/fetchers.ts`
- Studio in self-hosted mode (`IS_PLATFORM=false`) uses `alwaysLoggedIn` mode bypassing real auth entirely → `apps/studio/lib/auth.tsx`
- Client SDK's `fetchWithAuth` wraps every HTTP request, falling back to the anon key (`supabaseKey`) when no session exists, and sets both `apikey` and `Authorization: Bearer` headers → `packages/core/supabase-js/src/lib/fetch.ts`
- Client SDK auto-refreshes tokens every 30s, triggering refresh when expiry is within 90s; on TOKEN_REFRESHED event, it calls `realtime.setAuth(token)` to propagate the new JWT → `packages/core/auth-js/src/lib/constants.ts`
- All five services support the same JWT algorithm families (HS256/384/512, RS256/384/512, ES256/384/512, EdDSA) but Auth additionally filters JWKS to expose only asymmetric keys publicly → `internal/api/jwks.go`
- Storage's S3 Signature V4 path ultimately synthesizes a JWT from S3 credentials, converging with the JWT plugin so downstream code sees the same `request.jwt` and `request.jwtPayload` → `src/http/plugins/signature-v4.ts`

## Where It Appears

| Aspect | Auth (Go) | Storage (TS) | Realtime (Elixir) | Studio (React) | Client SDK (TS) |
|--------|-----------|-------------|-------------------|----------------|-----------------|
| **Token extraction** | Regex on Authorization header | Regex strip `Bearer ` prefix | WebSocket `apikey` param + `access_token` event | `getAccessToken()` from GoTrue session | `_getAccessToken()` from session or custom fn |
| **Verification lib** | `golang-jwt/jwt/v5` | `jose` (JS) | `Joken` (Elixir) | N/A (trusts platform API) | N/A (trusts Auth server) |
| **Algorithm support** | HS/RS/ES/Ed via config | HS/RS/ES/Ed from JWKS | HS/RS/ES/Ed via Joken signers | N/A | N/A |
| **Caching** | None (verifies per-request) | LRU cache 65K items, 50 MiB | None per-request, 5m re-auth interval | React Query for access token | localStorage + memory |
| **Fallback on missing JWT** | 401 Unauthorized | `role: 'anon'` if route allows | Connection rejected | Redirect to login | Falls back to anon key |
| **RLS integration** | N/A (is the issuer) | `set_config` injects claims into Postgres session | `set_conn_config` injects claims for RLS eval | N/A (uses platform API) | N/A (uses service APIs) |
| **Admin path** | JWT role in AdminRoles list | `apikey` header against admin keys | Bearer token against `api_jwt_secret` + blocklist | GoTrue session + permissions API | `accessToken` option for server-side |

## Design Intent

The pattern ensures a single source of truth for identity (Auth/GoTrue) while allowing each service to verify tokens independently without cross-service calls. This decoupled design means each service can scale and fail independently. The shared JWT structure acts as the integration contract -- all services agree on `sub`, `role`, `exp`, `aal` claims, enabling uniform RLS policy evaluation where applicable.

This pattern embodies two of Supabase's core product principles -> `sources/context/decisions/architecture-overview.md`:

- **Everything works in isolation**: Auth operates as a standalone JWT issuer backed only by Postgres — no external identity provider or service mesh is required. Each consuming service (Storage, Realtime, Studio) verifies tokens independently using only the shared signing secret, with no runtime dependency on Auth itself.
- **Everything is integrated**: The JWT is the universal integration contract. Auth issues it, and every other service consumes it through the same claim structure (`sub`, `role`, `exp`, `aal`). Each tool exposes an API that accepts this token, making cross-service composition possible without tight coupling.

## Trade-offs

- **Decoupled verification vs. token revocation latency**: Since each service verifies JWTs locally using the signing secret, a revoked session is not immediately reflected. Realtime mitigates this with periodic re-auth; Storage mitigates via short-lived cache TTLs.
- **Cache consistency**: Storage's JWT cache improves throughput but means a key rotation or secret change requires cache invalidation via PubSub.
- **Multiple auth paths in Storage**: Supporting REST JWT, S3 Signature V4, and TUS increases surface area but enables protocol flexibility.
- **Studio's dual mode**: `IS_PLATFORM` flag creates two distinct auth paths which could diverge in behavior.

## Agent Guidance

- When tracing an auth failure, first identify which service rejected the request. Auth returns 401 with error codes, Storage returns AccessDenied, Realtime closes the channel with shutdown_response.
- JWT claims propagation to Postgres RLS is done by both Storage (`TenantConnection.setScope`) and Realtime (`set_conn_config`) -- these set slightly different session variables, so RLS policies may reference different variable names.
- The Client SDK's `fetchWithAuth` is the single injection point for tokens into all sub-client HTTP requests (PostgREST, Storage, Functions). If auth is broken at this layer, all API calls fail.
- For self-hosted Studio (`IS_PLATFORM=false`), auth is entirely bypassed. Do not assume authentication is active in self-hosted environments.

## Related

- [[PROC-AUTH-AUTH]] -- Auth's internal authentication mechanism
- [[PROC-STORAGE-AUTH]] -- Storage authentication flow
- [[PROC-REALTIME-AUTH]] -- Realtime authentication flow
- [[PROC-STUDIO-AUTH]] -- Studio authentication mechanism
- [[PROC-CLIENT-SDK-AUTH]] -- Client SDK token lifecycle
