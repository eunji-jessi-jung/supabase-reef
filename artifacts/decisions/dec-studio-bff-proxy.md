---
id: "DEC-STUDIO-BFF-PROXY"
type: "decision"
title: "Studio BFF Proxy Pattern via Next.js API Routes"
domain: "studio"
status: "draft"
last_verified: 2026-04-15
freshness_note: "Deep read of BFF routes, fetchers, apiWrapper, apiHelpers, constants, self-hosted query on 2026-04-15"
freshness_triggers:
  - "apps/studio/pages/api/platform/"
  - "apps/studio/data/fetchers.ts"
  - "apps/studio/lib/api/apiWrapper.ts"
  - "apps/studio/lib/api/apiHelpers.ts"
  - "apps/studio/lib/api/apiAuthenticate.ts"
  - "apps/studio/lib/api/self-hosted/"
  - "apps/studio/lib/constants/index.ts"
known_unknowns:
  - "Original decision date and authors not available from code"
  - "Whether the team evaluated alternatives like tRPC or GraphQL federation before choosing raw Next.js API routes"
  - "Production latency overhead of the extra proxy hop per request"
tags:
  - decision
  - architecture
  - bff
  - nextjs
aliases: []
relates_to:
  - type: "parent"
    target: "[[SYS-STUDIO]]"
  - type: "refines"
    target: "[[API-STUDIO]]"
  - type: "refines"
    target: "[[PROC-STUDIO-AUTH]]"
sources:
  - category: "implementation"
    type: "github"
    ref: "apps/studio/pages/api/platform/pg-meta/[ref]/tables.ts"
  - category: "implementation"
    type: "github"
    ref: "apps/studio/data/fetchers.ts"
  - category: "implementation"
    type: "github"
    ref: "apps/studio/lib/api/apiWrapper.ts"
  - category: "implementation"
    type: "github"
    ref: "apps/studio/lib/api/apiHelpers.ts"
  - category: "implementation"
    type: "github"
    ref: "apps/studio/lib/api/apiAuthenticate.ts"
  - category: "implementation"
    type: "github"
    ref: "apps/studio/lib/constants/index.ts"
  - category: "implementation"
    type: "github"
    ref: "apps/studio/lib/api/self-hosted/query.ts"
  - category: "implementation"
    type: "github"
    ref: "apps/studio/lib/gotrue.ts"
  - category: "implementation"
    type: "github"
    ref: "apps/studio/data/query-client.ts"
  - category: "implementation"
    type: "github"
    ref: "apps/studio/next.config.ts"
  - category: "context"
    type: "documentation"
    ref: "sources/context/decisions/architecture-overview.md"
    notes: "Supabase product principle: 'everything is portable' — avoid lock-in, cloud compatible with self-hosted"
  - category: "context"
    type: "documentation"
    ref: "sources/context/decisions/self-hosting-overview.md"
    notes: "Self-hosting overview — Docker-based deployment, community-supported, same codebase as managed"
notes: ""
---

## Context

Supabase Studio is a dashboard that must communicate with many backend services: the platform management API, pg-meta (Postgres introspection), GoTrue (authentication), Storage, and others. The React client needs a unified, secure way to reach these services without exposing backend URLs, service keys, or connection strings to the browser. The dashboard also runs in two radically different modes -- Supabase-hosted platform (multi-project, GoTrue auth, external APIs) and self-hosted (single project, local DB, service key auth) -- requiring a layer that can adapt request routing and credential injection based on deployment mode.

## Decision

Route all backend communication through a Next.js Pages Router API route layer (`pages/api/`) that acts as a Backend-for-Frontend (BFF) proxy. The React client never calls backend services directly; instead, it sends requests to local `/api/platform/*` and `/api/v1/*` endpoints, which authenticate the user, inject server-side credentials, and forward requests to the appropriate backend.

## Key Facts

- 88 BFF route files under `pages/api/` organized into 10 backend service domains: auth, database, integrations, organizations, pg-meta, profile, projects, props, storage, telemetry -> `apps/studio/pages/api/platform/`
- Every BFF route handler is wrapped by `apiWrapper`, which provides global error catching and optional JWT authentication via `apiAuthenticate` before delegating to the route handler -> `apps/studio/lib/api/apiWrapper.ts`
- `apiWrapper` conditionally authenticates: when `IS_PLATFORM` is true and `withAuth` is requested, it calls `apiAuthenticate` to validate the bearer token via GoTrue's `getClaims`; in self-hosted mode, authentication is skipped entirely -> `apps/studio/lib/api/apiWrapper.ts`, `apps/studio/lib/api/apiAuthenticate.ts`
- Server-side header sanitization: `constructHeaders` in `apiHelpers.ts` strips all incoming headers except Accept, Authorization, Content-Type, cookie, and `x-connection-encrypted`, preventing browser headers (user-agent, referrer) from leaking to backend services -> `apps/studio/lib/api/apiHelpers.ts`
- Self-hosted mode injects `SUPABASE_SERVICE_KEY` as the `apiKey` header, allowing the single-project dashboard to authenticate without GoTrue -> `apps/studio/lib/api/apiHelpers.ts`
- The pg-meta proxy routes (tables, views, policies, triggers, etc.) construct redirect URLs to `PG_META_URL` while preserving query parameters; `PG_META_URL` resolves to `PLATFORM_PG_META_URL` in platform mode or `STUDIO_PG_META_URL` in self-hosted mode -> `apps/studio/pages/api/platform/pg-meta/[ref]/tables.ts`, `apps/studio/lib/constants/index.ts`
- The client-side `openapi-fetch` client in `fetchers.ts` adds two middleware layers: (1) auth middleware that injects `Authorization` and `X-Request-Id` headers on every request, and (2) a pg-meta connection guard that validates `x-connection-encrypted` is present before any `/platform/pg-meta/` call reaches the BFF, failing fast with a 400 -> `apps/studio/data/fetchers.ts`
- The `API_URL` constant resolves differently per environment: platform mode uses `NEXT_PUBLIC_API_URL`, browser self-hosted uses relative `/api`, Vercel preview uses `VERCEL_URL`, and test mode uses `localhost:3000` -> `apps/studio/lib/constants/index.ts`
- Self-hosted pg-meta query execution encrypts the local connection string and sends it via the `x-connection-encrypted` header, then proxies through the pg-meta service with Sentry span instrumentation -> `apps/studio/lib/api/self-hosted/query.ts`
- Legacy `fetchGet`/`fetchPost`/`fetchHeadWithTimeout` methods in `fetchers.ts` remain for dashboard-specific API endpoints not yet migrated to `openapi-fetch`, coexisting with the typed `client` -> `apps/studio/data/fetchers.ts`
- Error response middleware on the `openapi-fetch` client enriches error responses with `requestId`, `code`, `retryAfter`, and `requestPathname` before they reach React Query retry logic -> `apps/studio/data/fetchers.ts`
- React Query's `retryDelay` respects the `Retry-After` header from 429 responses, and specific high-load paths (`/run-lints`, `/usage`, `/query`, `/logs.all`) skip retries for non-429 errors to avoid amplifying load -> `apps/studio/data/query-client.ts`

## Rationale

The BFF proxy pattern serves several interrelated goals that would be difficult to achieve with direct client-to-service communication:

1. **Security**: Backend service URLs, API keys, and encrypted connection strings never reach the browser. The server-side layer strips and reconstructs headers, preventing credential leakage.
2. **Dual-mode deployment**: The `IS_PLATFORM` flag allows a single codebase to serve both Supabase Cloud (GoTrue JWT auth, multi-project, external API URLs) and self-hosted installs (service key auth, single project, local pg-meta) by swapping behavior at the proxy layer rather than in client code.
3. **Uniform error handling**: The `apiWrapper` global catch and `openapi-fetch` middleware ensure all backend errors are normalized into a consistent shape with request IDs, status codes, and retry metadata before reaching the UI.
4. **Type safety**: The `openapi-fetch` client is typed against auto-generated `api-types`, so all BFF route calls are compile-time checked, while the server-side proxy routes remain thin pass-throughs.
5. **Everything is portable**: The `IS_PLATFORM` dual-mode design directly supports Supabase's portability principle — Studio works both as the managed Supabase Cloud dashboard and as a self-hosted single-project dashboard deployed via Docker. The cloud offering is compatible with the self-hosted product, and users can migrate between them without changing their dashboard experience. -> `sources/context/decisions/architecture-overview.md`, `sources/context/decisions/self-hosting-overview.md`

## Consequences

### Positive
- Browser never holds service credentials; all secrets stay server-side
- Single API surface (`/api/platform/*`) abstracts away 10+ backend service endpoints
- `openapi-fetch` middleware provides cross-cutting auth, request tracing, and error normalization without per-route boilerplate
- Self-hosted and platform deployments share the same React client code; only the BFF layer adapts

### Negative
- Every request adds a proxy hop through the Next.js server, increasing latency
- 88 route files must be maintained as thin proxies, creating surface area for drift between the BFF layer and backend APIs
- Two fetching patterns coexist (typed `openapi-fetch` client and legacy `fetchGet`/`fetchPost`), increasing cognitive overhead during migration
- Pages Router API routes are a legacy Next.js pattern; migration to App Router route handlers would require rewriting the entire BFF layer

### Neutral
- The proxy pattern is invisible to end users; the dashboard URL structure does not expose the BFF indirection
- Error patterns (`error-patterns.ts`) provide a pluggable classification system for mapping backend error strings to typed error classes

## Related

- [[SYS-STUDIO]] -- parent system
- [[API-STUDIO]] -- full inventory of BFF routes and their backend targets
- [[PROC-STUDIO-AUTH]] -- authentication flow through apiWrapper and GoTrue
- [[GLOSSARY-STUDIO]] -- domain terminology (BFF, IS_PLATFORM, pg-meta)
