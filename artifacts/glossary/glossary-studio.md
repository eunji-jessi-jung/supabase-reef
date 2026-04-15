---
id: "GLOSSARY-STUDIO"
type: "glossary"
title: "Studio Domain Glossary"
domain: "studio"
status: "draft"
last_verified: 2026-04-15
freshness_note: "Deep read of data layer, BFF routes, fetchers, constants, pg-meta, api-types, query-client, next.config on 2026-04-15"
freshness_triggers:
  - "apps/studio/"
  - "packages/pg-meta/"
  - "packages/api-types/"
  - "apps/studio/data/fetchers.ts"
  - "apps/studio/lib/constants/index.ts"
  - "apps/studio/lib/api/"
  - "apps/studio/data/query-client.ts"
known_unknowns:
  - "Some terms may have business meanings not captured in code"
  - "Full list of Studio-specific SQL helper categories in pg-meta may grow"
tags:
  - glossary
aliases: []
relates_to:
  - type: "parent"
    target: "[[SYS-STUDIO]]"
  - type: "refines"
    target: "[[GLOSSARY-SUPABASE-REEF]]"
  - type: "refines"
    target: "[[DEC-STUDIO-BFF-PROXY]]"
sources:
  - category: "implementation"
    type: "code"
    ref: "apps/studio/package.json"
  - category: "implementation"
    type: "code"
    ref: "apps/studio/data/fetchers.ts"
  - category: "implementation"
    type: "code"
    ref: "apps/studio/lib/api/apiWrapper.ts"
  - category: "implementation"
    type: "code"
    ref: "apps/studio/lib/api/apiHelpers.ts"
  - category: "implementation"
    type: "code"
    ref: "apps/studio/lib/api/apiAuthenticate.ts"
  - category: "implementation"
    type: "code"
    ref: "apps/studio/lib/constants/index.ts"
  - category: "implementation"
    type: "code"
    ref: "apps/studio/data/query-client.ts"
  - category: "implementation"
    type: "code"
    ref: "apps/studio/data/database-policies/keys.ts"
  - category: "implementation"
    type: "code"
    ref: "apps/studio/data/database-policies/database-policies-query.ts"
  - category: "implementation"
    type: "code"
    ref: "apps/studio/lib/api/self-hosted/query.ts"
  - category: "implementation"
    type: "code"
    ref: "apps/studio/next.config.ts"
  - category: "implementation"
    type: "code"
    ref: "packages/pg-meta/src/index.ts"
  - category: "implementation"
    type: "code"
    ref: "packages/api-types/redocly.yaml"
  - category: "implementation"
    type: "code"
    ref: "pnpm-workspace.yaml"
  - category: "implementation"
    type: "code"
    ref: "turbo.jsonc"
  - category: "implementation"
    type: "code"
    ref: "apps/studio/lib/gotrue.ts"
  - category: "implementation"
    type: "code"
    ref: "apps/studio/data/error-patterns.ts"
notes: ""
---

## Terms

| Term | Definition | Code Location | See Also |
|------|-----------|---------------|----------|
| Project | A Supabase project instance -- the primary organizational unit. Each project has its own database, auth config, storage buckets, and API keys. In platform mode, users manage multiple projects; in self-hosted mode, all traffic is redirected to `project/default`. | `apps/studio/data/projects/`, `apps/studio/next.config.ts` (redirects) | [[GLOSSARY-SUPABASE-REEF]] |
| pg-meta | TypeScript library for Postgres system catalog introspection. Generates parameterized SQL against `pg_catalog` and `information_schema`. Zero runtime dependencies beyond Zod. Exports 18 introspection modules plus Studio-specific SQL helpers for advisor, auth, storage, database, table-editor, sql-editor, role-impersonation, and integrations. | `packages/pg-meta/src/index.ts`, `packages/pg-meta/src/pg-meta-*.ts` | [[SCH-STUDIO-PG-META]] |
| api-types | Shared TypeScript type definitions auto-generated from two OpenAPI endpoints (v1 API and platform API) via `openapi-typescript` and `redocly.yaml`. Produces `api.d.ts` and `platform.d.ts`. Consumed by `openapi-fetch` client for compile-time type safety. | `packages/api-types/types/`, `packages/api-types/redocly.yaml` | |
| BFF | Backend-for-Frontend -- the Next.js Pages Router API routes (`pages/api/`) that proxy all requests from the React client to backend services. Adds authentication, header sanitization, credential injection, and error normalization. The React client never directly calls backend services. | `apps/studio/pages/api/platform/`, `apps/studio/lib/api/apiWrapper.ts` | [[DEC-STUDIO-BFF-PROXY]] |
| apiWrapper | Global catch-all wrapper function for every BFF route handler. Provides error handling and optional JWT authentication (when `IS_PLATFORM && withAuth`). Returns 401 on auth failure, 500 on unhandled exceptions. | `apps/studio/lib/api/apiWrapper.ts` | |
| apiAuthenticate | Server-side function that validates the bearer JWT token by calling GoTrue's `getClaims`. Extracts the token from the `Authorization` header and returns JWT claims on success. Used by `apiWrapper` when authentication is required. | `apps/studio/lib/api/apiAuthenticate.ts` | [[PROC-STUDIO-AUTH]] |
| IS_PLATFORM | Boolean flag (`NEXT_PUBLIC_IS_PLATFORM === 'true'`) that switches Studio between Supabase-hosted platform mode (GoTrue auth, external API URLs, multi-project) and self-hosted mode (service key auth, local DB, single project `project/default`). Controls auth behavior, API URL resolution, redirects, CSP, Sentry, and React Query online status. | `apps/studio/lib/constants/index.ts` | |
| openapi-fetch | Type-safe HTTP client created from `api-types` that provides compile-time checking for every API call. Uses middleware for auth header injection, request ID generation, pg-meta connection guards, and error response enrichment. | `apps/studio/data/fetchers.ts` | |
| Data hooks | React Query hooks organized by domain (~97 subdirectories under `data/`). Follow a consistent pattern: `keys.ts` for query key factories, `*-query.ts` for read hooks, `*-mutation.ts` for write hooks. Each query hook uses `queryOptions` pattern with TanStack Query v5. | `apps/studio/data/`, e.g. `apps/studio/data/database-policies/` | |
| Query keys | Hierarchical arrays used by React Query for cache management, defined in `keys.ts` files. Typically structured as `['projects', projectRef, '{domain}', ...filters]`. Enable granular cache invalidation per project and domain. | `apps/studio/data/database-policies/keys.ts` (example) | |
| Branch | Database branching feature -- creates a copy of a project's database for development/preview. | `apps/studio/data/branches/` | |
| SQL Editor | Studio feature for writing and executing SQL queries against a project's database. Queries are submitted through the BFF pg-meta proxy via `POST /platform/pg-meta/{ref}/query`. | `apps/studio/data/sql/`, `apps/studio/pages/api/platform/pg-meta/[ref]/query/index.ts` | |
| Table Editor | Visual interface for managing database tables, columns, and relationships without writing SQL. Uses pg-meta introspection for metadata and Studio-specific SQL helpers for mutations. | `apps/studio/data/table-editor/`, `packages/pg-meta/src/sql/studio/table-editor` | |
| RLS | Row Level Security -- Postgres feature for row-level access control. Studio provides a visual policy editor backed by the pg-meta policies module. | `apps/studio/data/database-policies/`, `packages/pg-meta/src/pg-meta-policies.ts` | [[GLOSSARY-SUPABASE-REEF]] |
| x-connection-encrypted | HTTP header carrying the encrypted Postgres connection string. Required for all `/platform/pg-meta/` requests in platform mode. The `openapi-fetch` middleware validates its presence before requests leave the browser (connection guard). In self-hosted mode, the connection string is encrypted server-side via `encryptString`. | `apps/studio/data/fetchers.ts` (guard), `apps/studio/lib/api/self-hosted/query.ts` (encryption) | |
| PG_META_URL | Server-side constant pointing to the pg-meta backend service. Resolves to `PLATFORM_PG_META_URL` in platform mode or `STUDIO_PG_META_URL` in self-hosted mode. BFF routes use it to construct proxy target URLs. | `apps/studio/lib/constants/index.ts` | |
| API_URL | Client-side constant for the BFF base URL. Resolves differently per environment: `NEXT_PUBLIC_API_URL` (platform), `/api` (browser self-hosted), `VERCEL_URL` (Vercel preview), `localhost:3000/api` (test). | `apps/studio/lib/constants/index.ts` | |
| constructHeaders (apiHelpers) | Server-side header sanitization function. Strips all incoming browser headers except Accept, Authorization, Content-Type, cookie, and `x-connection-encrypted`. In self-hosted mode, injects `SUPABASE_SERVICE_KEY` as `apiKey`. Not to be confused with `constructHeaders` in `fetchers.ts` (client-side). | `apps/studio/lib/api/apiHelpers.ts` | |
| constructHeaders (fetchers) | Client-side function that builds request headers with a generated `X-Request-Id` UUID and Bearer token from `getAccessToken()`. Used by the `openapi-fetch` middleware and legacy fetch helpers. | `apps/studio/data/fetchers.ts` | |
| ResponseError | Base error class for API failures. Carries `message`, `code` (HTTP status), `requestId`, `retryAfter`, `requestPathname`, and optional `metadata`. Thrown by `handleError` and matched against `ERROR_PATTERNS` for classification into typed subclasses. | `apps/studio/data/fetchers.ts`, `apps/studio/data/error-patterns.ts` | |
| ERROR_PATTERNS | Pluggable error classification system mapping regex patterns to typed error subclasses (e.g., `ConnectionTimeoutError`). Used by `handleError` to convert generic API errors into domain-specific error types for UI display. | `apps/studio/data/error-patterns.ts` | |
| Turborepo | Build orchestrator for the monorepo. Defines task dependencies (`build -> ^build`, `typecheck -> ^typecheck`), caches `dist/**` and `.next/**` outputs, and provides 14-day remote cache. Studio builds are triggered via `turbo run build --filter=studio`. | `turbo.jsonc`, `package.json` (root) | |
| pnpm workspace | Package manager workspace configuration using pnpm 10. Defines workspace globs (`apps/*`, `packages/*`, `blocks/*`, `e2e/*`) and a version catalog for shared dependency versions (Next.js 16.2.3, React 18, TypeScript 6.0, Zod 3.25). | `pnpm-workspace.yaml` | |
| standalone output | Next.js build output mode (`output: 'standalone'`) used by Studio for containerized deployment. Produces a self-contained build that can run without `node_modules`, used by the Docker build target. | `apps/studio/next.config.ts` | |
| GoTrue | Supabase authentication service. Studio uses GoTrue via `@supabase/auth-js` (through the `common/gotrue` package) for JWT validation on the server side (`getUserClaims`) and session management on the client side. | `apps/studio/lib/gotrue.ts`, `packages/common/` | [[SYS-AUTH]] |

## Disambiguation

- **constructHeaders**: Two distinct functions share this name. The server-side version in `apiHelpers.ts` sanitizes incoming browser headers for BFF proxy forwarding. The client-side version in `fetchers.ts` builds outgoing headers with auth tokens and request IDs. They serve opposite ends of the same request flow.
- **pg-meta (library vs service)**: `@supabase/pg-meta` is a TypeScript SQL-generation library with zero runtime deps. The pg-meta *service* is a separate backend that executes those queries against Postgres. Studio's BFF routes proxy to the service, while Studio's client code imports the library directly for types and constants.
- **fetchGet/fetchPost vs openapi-fetch client**: Two fetching patterns coexist. The `openapi-fetch` typed client (via `get`, `post`, `del` exports from `fetchers.ts`) is the modern approach with compile-time type safety. The legacy `fetchGet`/`fetchPost` functions remain for dashboard-specific endpoints not yet migrated. Both share the same `constructHeaders` (client-side) for auth.

## Naming Conventions

- Data modules: `apps/studio/data/{domain}/` with `keys.ts` for query key factories, `*-query.ts` for reads, `*-mutation.ts` for writes (e.g., `database-policy-create-mutation.ts`)
- BFF routes: `apps/studio/pages/api/platform/{service}/[ref]/` with each route file handling a specific resource type
- Component structure: `apps/studio/components/interfaces/` for feature-specific components
- Package naming: `@supabase/{package-name}` for npm-published packages
- Workspace catalog: shared dependency versions defined in `pnpm-workspace.yaml` under `catalog:` and referenced as `"catalog:"` in package.json files

## Related

- [[SYS-STUDIO]] -- parent system
- [[DEC-STUDIO-BFF-PROXY]] -- architectural decision for the BFF proxy pattern
- [[SCH-STUDIO-PG-META]] -- pg-meta introspection model and entity catalog
- [[GLOSSARY-SUPABASE-REEF]] -- unified cross-service glossary
