---
id: "PROC-CLIENT-SDK-AUTH"
type: "process"
title: "Client SDK Authentication and Token Lifecycle"
domain: "supabase-reef"
status: draft
last_verified: 2026-04-15
freshness_note: "Deep scan of SupabaseClient, fetchWithAuth, GoTrueClient (auth-js), locks, constants, and SupabaseAuthClient"
freshness_triggers:
  - "packages/core/auth-js/src/GoTrueClient.ts"
  - "packages/core/auth-js/src/lib/constants.ts"
  - "packages/core/auth-js/src/lib/errors.ts"
  - "packages/core/auth-js/src/lib/fetch.ts"
  - "packages/core/auth-js/src/lib/local-storage.ts"
  - "packages/core/auth-js/src/lib/locks.ts"
  - "packages/core/supabase-js/src/SupabaseClient.ts"
  - "packages/core/supabase-js/src/lib/SupabaseAuthClient.ts"
  - "packages/core/supabase-js/src/lib/fetch.ts"
known_unknowns:
  - "Exact behavior of BroadcastChannel cross-tab session sync — visible in GoTrueClient but complex"
  - "How the auth lock steal recovery interacts with React Strict Mode double-mount in production"
tags:
  - client-sdk
  - auth
  - jwt
  - gotrue
  - token-refresh
aliases:
  - "SDK Auth"
  - "supabase-js Auth"
relates_to:
  - type: "depends_on"
    target: "[[SYS-AUTH]]"
  - type: "integrates_with"
    target: "[[PROC-STUDIO-AUTH]]"
  - type: "parent"
    target: "[[SYS-CLIENT-SDK]]"
sources:
  - category: implementation
    type: github
    ref: "packages/core/auth-js/src/GoTrueClient.ts"
    notes: "Core auth client with session management, token refresh, state change events"
  - category: implementation
    type: github
    ref: "packages/core/auth-js/src/lib/constants.ts"
    notes: "AUTO_REFRESH_TICK_DURATION_MS (30s), EXPIRY_MARGIN_MS, NETWORK_FAILURE retry config"
  - category: implementation
    type: github
    ref: "packages/core/auth-js/src/lib/errors.ts"
    notes: "Full auth error hierarchy: AuthError, AuthApiError, AuthSessionMissingError, etc."
  - category: implementation
    type: github
    ref: "packages/core/auth-js/src/lib/fetch.ts"
    notes: "Auth-specific fetch with retryable error handling and API version headers"
  - category: implementation
    type: github
    ref: "packages/core/auth-js/src/lib/local-storage.ts"
    notes: "memoryLocalStorageAdapter fallback for non-browser environments"
  - category: implementation
    type: github
    ref: "packages/core/auth-js/src/lib/locks.ts"
    notes: "Navigator Lock API and process-level lock implementations for concurrency control"
  - category: implementation
    type: github
    ref: "packages/core/supabase-js/src/SupabaseClient.ts"
    notes: "Orchestrates auth initialization, token injection, and Realtime re-auth"
  - category: implementation
    type: github
    ref: "packages/core/supabase-js/src/lib/SupabaseAuthClient.ts"
    notes: "Thin wrapper extending AuthClient (GoTrueClient)"
  - category: implementation
    type: github
    ref: "packages/core/supabase-js/src/lib/fetch.ts"
    notes: "fetchWithAuth wrapper that injects apikey and Bearer token"
notes: ""
---

## Purpose

Documents how the Client SDK manages authentication tokens and injects them into requests across all sub-clients. The SDK acts as a token orchestrator — it obtains JWTs from GoTrue, stores them, refreshes them automatically, and injects them into PostgREST, Storage, Realtime, and Functions requests.

## Key Facts

- `fetchWithAuth` wraps every HTTP request: it calls `getAccessToken()` to get the current JWT, falls back to the `supabaseKey` (anon key) if no session exists, and sets both `apikey` and `Authorization: Bearer` headers → `packages/core/supabase-js/src/lib/fetch.ts`
- `SupabaseAuthClient` is a thin subclass of `AuthClient` (which is just `GoTrueClient`); no additional logic is added — all behavior comes from GoTrueClient → `packages/core/supabase-js/src/lib/SupabaseAuthClient.ts`
- `_getAccessToken()` on SupabaseClient checks for a custom `accessToken` function first (for server-side use), then falls back to `auth.getSession()` extracting `session.access_token`, then to the raw `supabaseKey` → `packages/core/supabase-js/src/SupabaseClient.ts`
- When `accessToken` option is provided (server-side), `supabase.auth` is replaced with a Proxy that throws on any property access, preventing accidental client-side auth method calls → `packages/core/supabase-js/src/SupabaseClient.ts`
- Auto-refresh checks run every 30 seconds (`AUTO_REFRESH_TICK_DURATION_MS = 30_000`); refresh is triggered when the token will expire within 3 ticks (90 seconds, `EXPIRY_MARGIN_MS`) → `packages/core/auth-js/src/lib/constants.ts`
- Network failure retry uses up to 10 retries with 200ms intervals (2 deciseconds) for transient fetch errors; HTTP status codes 502, 503, 504, 520-524, 530 are treated as retryable network errors → `packages/core/auth-js/src/lib/constants.ts` and `packages/core/auth-js/src/lib/fetch.ts`
- Session persistence defaults to `localStorage` with key `sb-{project-ref}-auth-token`; for non-browser environments, a `memoryLocalStorageAdapter` provides an in-memory fallback; React Native users can supply `AsyncStorage` or encrypted `SecureStore` via the `storage` option → `packages/core/supabase-js/src/SupabaseClient.ts` and `packages/core/auth-js/src/lib/local-storage.ts`
- Token refresh uses the Navigator Lock API (`navigatorLock`) for concurrency control across tabs; if a lock times out (e.g., React Strict Mode orphaned lock), it forcefully steals the lock to recover → `packages/core/auth-js/src/lib/locks.ts`
- For non-browser single-process environments (React Native), `processLock` provides equivalent concurrency control using a promise chain with timeout detection → `packages/core/auth-js/src/lib/locks.ts`
- Auth state changes fire via `onAuthStateChange` with events: `SIGNED_IN`, `SIGNED_OUT`, `TOKEN_REFRESHED`, `USER_UPDATED`, `PASSWORD_RECOVERY`; SupabaseClient listens and calls `realtime.setAuth(token)` on `TOKEN_REFRESHED` and `SIGNED_IN` → `packages/core/supabase-js/src/SupabaseClient.ts`
- Cross-tab session sync uses `BroadcastChannel` keyed by `storageKey`, enabled only when `persistSession` is true and the API is available → `packages/core/auth-js/src/GoTrueClient.ts`
- On `SIGNED_OUT` from storage events (another tab signed out), SupabaseClient calls `this.auth.signOut()` to clear the local session and resets the Realtime auth → `packages/core/supabase-js/src/SupabaseClient.ts`
- If auto-refresh fails with a non-retryable error (e.g., invalid refresh token), the session is removed entirely via `_removeSession()` → `packages/core/auth-js/src/GoTrueClient.ts`
- All auth HTTP requests include `X-Supabase-Api-Version: 2024-01-01` and `X-Client-Info: gotrue-js/{version}` headers → `packages/core/auth-js/src/lib/fetch.ts` and `packages/core/auth-js/src/lib/constants.ts`

## Steps

1. **Client creation** — `createClient(url, key)` initializes SupabaseClient; generates storage key `sb-{hostname}-auth-token`; creates GoTrueClient with auto-refresh and persist-session defaults
2. **fetchWithAuth binding** — `fetchWithAuth(supabaseKey, _getAccessToken, customFetch)` is created and passed to all sub-clients (PostgREST, Storage, Functions)
3. **Session initialization** — GoTrueClient attempts to load existing session from storage; if `detectSessionInUrl` is true, checks URL for OAuth callback parameters
4. **Sign in** — User calls `supabase.auth.signInWithPassword(...)` or other sign-in method; GoTrueClient stores access_token and refresh_token; fires `SIGNED_IN` event
5. **Token injection** — On every sub-client HTTP request, `fetchWithAuth` calls `_getAccessToken()` which reads the current session's access_token
6. **Auto-refresh tick** — Every 30s, `_autoRefreshTokenTick` checks if token expires within 90s; if so, acquires a navigator lock and calls `_callRefreshToken(refresh_token)`
7. **State propagation** — `onAuthStateChange` fires with event type; SupabaseClient's listener calls `realtime.setAuth(token)` to update WebSocket auth
8. **Cross-tab sync** — BroadcastChannel relays auth events to other tabs; receiving tabs call `_notifyAllSubscribers` without re-broadcasting
9. **Sign out** — `signOut()` clears session from storage, fires `SIGNED_OUT`, Realtime auth is cleared via `realtime.setAuth()`

## Worked Examples

### Auto-Refresh Preventing Expired Requests

A user's JWT expires at T+3600s. At T+3510s (90s before expiry), the auto-refresh tick detects the token falls within the `EXPIRY_MARGIN_MS` window. It acquires a navigator lock, calls the GoTrue `/token?grant_type=refresh_token` endpoint, receives a new JWT. The `TOKEN_REFRESHED` event fires, SupabaseClient updates Realtime auth, and the next PostgREST request uses the fresh token automatically via `fetchWithAuth`.

### Server-Side Access Token Mode

On a Next.js server component, `createClient` is called with `accessToken: async () => getServerToken()`. The `auth` property becomes a Proxy — any call to `supabase.auth.getSession()` throws immediately, preventing server-side misuse. All sub-client requests use the custom `accessToken` function to get the Bearer token.

## Related

- [[SYS-CLIENT-SDK]] — parent system
- [[SYS-AUTH]] — GoTrue issues and refreshes JWTs
- [[PROC-STUDIO-AUTH]] — Studio uses similar GoTrue client for platform auth
