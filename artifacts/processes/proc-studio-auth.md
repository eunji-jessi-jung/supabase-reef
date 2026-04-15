---
id: "PROC-STUDIO-AUTH"
type: "process"
title: "Studio Authentication and Authorization Mechanism"
domain: "supabase-reef"
status: draft
last_verified: 2026-04-15
freshness_note: "Deep scan of auth data hooks, lib/auth.tsx, fetchers middleware, permissions, and access token modules"
freshness_triggers:
  - "apps/studio/data/access-tokens/"
  - "apps/studio/data/api-authorization/"
  - "apps/studio/data/auth/"
  - "apps/studio/data/fetchers.ts"
  - "apps/studio/data/permissions/"
  - "apps/studio/lib/auth.tsx"
  - "apps/studio/lib/constants/index.ts"
known_unknowns:
  - "Exact cookie-based session storage mechanism — the GoTrue client is imported from 'common' package and its storage config is not visible in Studio code"
  - "How Studio platform auth interacts with Supabase's hosted GoTrue instance (sign-up, MFA flows)"
  - "Full OAuth scope model for third-party API authorization approvals"
tags:
  - studio
  - auth
  - gotrue
  - platform-auth
aliases:
  - "Studio Auth"
  - "Dashboard Authentication"
relates_to:
  - type: "parent"
    target: "[[SYS-STUDIO]]"
  - type: "depends_on"
    target: "[[SYS-AUTH]]"
  - type: "integrates_with"
    target: "[[PROC-CLIENT-SDK-AUTH]]"
sources:
  - category: implementation
    type: github
    ref: "apps/studio/data/access-tokens/access-tokens-query.ts"
    notes: "Platform access token CRUD via /platform/profile/access-tokens"
  - category: implementation
    type: github
    ref: "apps/studio/data/api-authorization/api-authorization-query.ts"
    notes: "OAuth authorization approval flow with scopes"
  - category: implementation
    type: github
    ref: "apps/studio/data/auth/keys.ts"
    notes: "React Query key factory for all auth-related queries"
  - category: implementation
    type: github
    ref: "apps/studio/data/auth/session-access-token-query.ts"
    notes: "Session access token retrieval hook"
  - category: implementation
    type: github
    ref: "apps/studio/data/fetchers.ts"
    notes: "Central HTTP client with auth middleware and error handling"
  - category: implementation
    type: github
    ref: "apps/studio/data/permissions/permissions-query.ts"
    notes: "Role-based permission checks"
  - category: implementation
    type: github
    ref: "apps/studio/lib/auth.tsx"
    notes: "AuthProvider wrapper, sign-out hook, GoTrue error handling"
  - category: implementation
    type: github
    ref: "apps/studio/lib/constants/index.ts"
    notes: "IS_PLATFORM flag and GOTRUE_ERRORS constants"
notes: ""
---

## Purpose

Documents how Studio authenticates dashboard users and authorizes their access to project resources. Studio operates at two distinct auth levels: **platform auth** (authenticating the developer to the Supabase dashboard via GoTrue) and **project auth** (authorizing API operations on specific Supabase projects via access tokens and permissions).

## Key Facts

- Studio wraps GoTrue via `AuthProviderInternal` from the `common` package; when `IS_PLATFORM` is false (local/self-hosted), it uses `alwaysLoggedIn` mode bypassing real auth entirely → `apps/studio/lib/auth.tsx`
- `IS_PLATFORM` is set by the `NEXT_PUBLIC_IS_PLATFORM` environment variable and gates all platform-specific auth behavior including permissions queries and API URL resolution → `apps/studio/lib/constants/index.ts`
- `gotrueClient` (imported from `common`) handles sign-in, sign-out, and session management; `useSignOut` clears GoTrue session, PostHog identity, localStorage, AI assistant IndexedDB, and React Query cache → `apps/studio/lib/auth.tsx`
- Auth errors surface via `useAuthError` hook and display as toast notifications using `sonner`; special handling exists for `UNVERIFIED_GITHUB_USER` GoTrue errors directing users to verify email on GitHub → `apps/studio/lib/auth.tsx`
- Session access tokens are retrieved via `getAccessToken()` from `common` package, wrapped in a React Query hook with key `['access-token']`; returns empty string server-side (SSR guard) → `apps/studio/data/auth/session-access-token-query.ts`
- The central `openapi-fetch` client injects auth via middleware: `constructHeaders()` calls `getAccessToken()` and sets both `Authorization: Bearer` header and a unique `X-Request-Id` per request → `apps/studio/data/fetchers.ts`
- The HTTP client uses `credentials: 'include'` on all requests, indicating cookie-based session transport alongside Bearer tokens → `apps/studio/data/fetchers.ts`
- Platform access tokens (PATs) are managed via dedicated CRUD hooks at `/platform/profile/access-tokens` endpoint, separate from session tokens → `apps/studio/data/access-tokens/access-tokens-query.ts`
- Permissions are fetched from `/platform/profile/permissions` with a 5-minute stale time, gated by `IS_PLATFORM && isLoggedIn`; they include actions, conditions (json-logic), organization slugs, and project refs → `apps/studio/data/permissions/permissions-query.ts`
- OAuth API authorization supports approve/decline flows with scopes (OAuthScope type), tracking app name, domain, icon, and expiration — used for third-party integrations → `apps/studio/data/api-authorization/api-authorization-query.ts`
- A `pgMetaGuard` middleware intercepts `/platform/pg-meta/` requests and throws a 400 `ResponseError` if no valid `x-connection-encrypted` header is present, preventing unnecessary backend hops → `apps/studio/data/fetchers.ts`
- The auth query key factory in `authKeys` organizes queries by user, users-infinite, users-count, auth-config, access-token, and overview-metrics — all namespaced under `['projects', projectRef, ...]` except access-token → `apps/studio/data/auth/keys.ts`
- Permissions query result is force-cast with a TODO noting the API response type is not properly typed: `return data as unknown as PermissionsResponse` → `apps/studio/data/permissions/permissions-query.ts`

## Steps

1. **Platform login** — Developer authenticates via GoTrue (hosted by Supabase platform); `AuthProviderInternal` initializes session and sets cookies
2. **Session token retrieval** — `getAccessToken()` extracts the current JWT from the GoTrue session; this is cached via React Query
3. **Request middleware** — Every API call through the `openapi-fetch` client runs `constructHeaders()` middleware which attaches `Authorization: Bearer {token}` and `X-Request-Id`
4. **Permissions check** — On platform, `usePermissionsQuery` fetches the user's permission set (actions + json-logic conditions + resource scopes) from `/platform/profile/permissions`
5. **Project-level operations** — Requests to project APIs include project ref; pg-meta requests additionally require `x-connection-encrypted` header validated by `pgMetaGuard`
6. **Token refresh** — GoTrue client handles automatic token refresh; on auth errors, toasts notify the user
7. **Sign out** — `useSignOut` orchestrates: GoTrue sign-out, PostHog reset, localStorage clear, IndexedDB clear, React Query cache clear

## Worked Examples

### Platform vs Self-Hosted Auth Flow

On the hosted platform (`IS_PLATFORM=true`), a developer hits the Studio login page, authenticates via GoTrue (email/password or GitHub OAuth), and receives a session with JWT. The `AuthProvider` wraps the app and makes the session available. Every subsequent API call injects the JWT automatically.

In self-hosted mode (`IS_PLATFORM=false`), `AuthProviderInternal` runs with `alwaysLoggedIn=true`, bypassing GoTrue entirely. Permissions queries are disabled. The developer accesses Studio directly without authentication.

## Related

- [[SYS-STUDIO]] — parent system
- [[SYS-AUTH]] — provides GoTrue infrastructure for platform auth
- [[PROC-CLIENT-SDK-AUTH]] — client-side auth token management pattern
