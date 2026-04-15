---
id: "DEC-CLIENT-SDK-AUTH-TOKEN-INJECTION"
type: "decision"
title: "Cross-Cutting Auth Token Injection via fetchWithAuth"
domain: "client-sdk"
status: "draft"
last_verified: 2026-04-15
freshness_note: "scuba-depth trace of SupabaseClient constructor and fetch.ts"
freshness_triggers:
  - "packages/core/supabase-js/src/lib/fetch.ts"
  - "packages/core/supabase-js/src/SupabaseClient.ts"
  - "packages/core/supabase-js/src/lib/types.ts"
  - "packages/core/auth-js/src/GoTrueClient.ts"
known_unknowns:
  - "Whether the fetchWithAuth approach has measurable latency cost from calling getSession on every request"
  - "How custom accessToken provider interacts with token caching under concurrent requests"
  - "Original ADR or discussion thread that led to this design (not in code)"
tags:
  - decision
  - architecture
  - auth
  - client-sdk
aliases:
  - "fetchWithAuth"
  - "access token injection"
relates_to:
  - type: "parent"
    target: "[[SYS-CLIENT-SDK]]"
  - type: "related"
    target: "[[API-CLIENT-SDK]]"
  - type: "related"
    target: "[[PROC-CLIENT-SDK-AUTH]]"
  - type: "related"
    target: "[[GLOSSARY-CLIENT-SDK]]"
sources:
  - category: "implementation"
    type: "code"
    ref: "packages/core/supabase-js/src/lib/fetch.ts"
  - category: "implementation"
    type: "code"
    ref: "packages/core/supabase-js/src/SupabaseClient.ts"
  - category: "implementation"
    type: "code"
    ref: "packages/core/supabase-js/src/lib/types.ts"
  - category: "implementation"
    type: "code"
    ref: "packages/core/supabase-js/src/lib/constants.ts"
  - category: "implementation"
    type: "code"
    ref: "packages/core/functions-js/src/FunctionsClient.ts"
notes: ""
---

## Context

The Supabase JS SDK composes five independent sub-clients (auth, postgrest, realtime, storage, functions) into a single SupabaseClient. Each sub-client is a standalone npm package that makes HTTP or WebSocket requests to different backend services. All of these requests must carry a valid auth token (either the user's session JWT or the anon API key) so that Supabase's Row-Level Security and service authorization work correctly. The question is: how should the SDK ensure every sub-client request carries the correct, current auth token without tightly coupling each sub-client to the auth system?

## Decision

Inject a single `fetchWithAuth` wrapper as the custom `fetch` implementation for all HTTP-based sub-clients. This wrapper intercepts every outgoing request and lazily resolves the current access token from the auth session (or from a user-supplied `accessToken` callback). The Realtime (WebSocket) sub-client uses a separate `setAuth()` mechanism driven by auth state change events.

## Key Facts

- `fetchWithAuth` is a higher-order function that wraps any `fetch` implementation, adding `apikey` and `Authorization` headers on every call → `packages/core/supabase-js/src/lib/fetch.ts`
- The wrapper calls `getAccessToken()` on every request to get the latest token, falling back to the anon `supabaseKey` if no session exists → `packages/core/supabase-js/src/lib/fetch.ts` lines 22-23
- Headers are set only if not already present, allowing callers to override auth headers → `packages/core/supabase-js/src/lib/fetch.ts` lines 25-29
- SupabaseClient binds `fetchWithAuth` once in the constructor and passes it to PostgREST, Storage, and Functions sub-clients → `packages/core/supabase-js/src/SupabaseClient.ts` line 327
- The `_getAccessToken()` method supports two modes: (1) built-in auth via `this.auth.getSession()`, or (2) external third-party auth via a user-supplied `accessToken` callback → `packages/core/supabase-js/src/SupabaseClient.ts` lines 533-541
- When `accessToken` option is set, the auth sub-client is replaced with a Proxy that throws on any access, preventing accidental use of Supabase Auth alongside third-party auth → `packages/core/supabase-js/src/SupabaseClient.ts` lines 316-325
- The Realtime sub-client receives tokens differently: auth state changes trigger `_handleTokenChanged()` which calls `realtime.setAuth(token)` → `packages/core/supabase-js/src/SupabaseClient.ts` lines 592-615
- `TOKEN_REFRESHED` and `SIGNED_IN` events push the new token to Realtime; `SIGNED_OUT` clears it and optionally triggers signout on the auth client → `packages/core/supabase-js/src/SupabaseClient.ts` lines 599-615
- Users can supply a fully custom `fetch` via `options.global.fetch`, which `fetchWithAuth` wraps rather than replaces → `packages/core/supabase-js/src/lib/fetch.ts` lines 3-8 and `SupabaseClient.ts` line 327
- FunctionsClient is instantiated fresh on every `.functions` property access (getter pattern), receiving the pre-configured `this.fetch` each time → `packages/core/supabase-js/src/SupabaseClient.ts` lines 364-369
- The `changedAccessToken` field deduplicates token updates to Realtime, preventing unnecessary `setAuth` calls when the token hasn't actually changed → `packages/core/supabase-js/src/SupabaseClient.ts` lines 606-609
- For external `accessToken` mode, the initial Realtime token is set eagerly via `Promise.resolve(this.accessToken()).then(...)` to avoid race conditions with early channel subscriptions → `packages/core/supabase-js/src/SupabaseClient.ts` lines 333-339

## Rationale

The fetchWithAuth approach achieves several goals simultaneously:
1. **Decoupling**: Sub-client packages (postgrest-js, storage-js, functions-js) have zero knowledge of auth-js. They accept a standard `fetch` and the auth concern is handled transparently.
2. **Lazy token resolution**: Tokens are resolved at request time, not at client construction time. This means token refreshes are automatically picked up without re-initializing sub-clients.
3. **Custom fetch compatibility**: The wrapper composes with user-supplied `fetch` implementations (needed for environments like Cloudflare Workers, React Native, etc.) rather than fighting them.
4. **Third-party auth support**: The `accessToken` callback option cleanly supports non-Supabase auth providers (Firebase Auth, Auth0, Clerk, etc.) through the same injection path.

## Consequences

### Positive
- Sub-client packages remain independently usable without any Supabase auth dependency
- Token refresh is transparent to application code; no manual header management needed
- A single point of change if auth header format ever changes
- Custom fetch implementations (for edge runtimes, proxies, logging) compose naturally

### Negative
- Every HTTP request incurs an async `getAccessToken()` call, which reads from the auth session store
- The Realtime sub-client needs a separate push-based token delivery mechanism (event listener + `setAuth()`), creating two distinct auth injection patterns in one SDK
- The Proxy-based auth lockout when using external `accessToken` can be confusing at runtime (throws on any `.auth.*` property access)

### Neutral
- The pattern is invisible to end users; they never interact with `fetchWithAuth` directly
- FunctionsClient being re-instantiated on every `.functions` access is a minor allocation cost but ensures fresh headers

## Related

- [[SYS-CLIENT-SDK]] -- parent system
- [[API-CLIENT-SDK]] -- public API surface affected by this decision
- [[PROC-CLIENT-SDK-AUTH]] -- auth flow procedures that depend on this injection
- [[GLOSSARY-CLIENT-SDK]] -- terms defined by this pattern
