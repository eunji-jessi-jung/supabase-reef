---
id: "GLOSSARY-CLIENT-SDK"
type: "glossary"
title: "Client SDK Domain Glossary"
domain: "client-sdk"
status: "draft"
last_verified: 2026-04-15
freshness_note: "scuba-depth glossary from artifact and code scan"
freshness_triggers:
  - "packages/core/supabase-js/src/SupabaseClient.ts"
  - "packages/core/supabase-js/src/lib/fetch.ts"
  - "packages/core/supabase-js/src/lib/types.ts"
  - "packages/core/supabase-js/src/lib/constants.ts"
  - "packages/core/supabase-js/src/index.ts"
  - "packages/core/auth-js/src/GoTrueClient.ts"
  - "packages/core/postgrest-js/src/PostgrestClient.ts"
  - "packages/core/realtime-js/src/RealtimeClient.ts"
  - "packages/core/realtime-js/src/RealtimeChannel.ts"
  - "packages/core/storage-js/src/StorageClient.ts"
  - "packages/core/functions-js/src/FunctionsClient.ts"
known_unknowns:
  - "Internal terminology used by the Supabase team that does not appear in code (e.g., 'banana split' or other code names)"
  - "Full list of GenericSchema sub-types and their intended usage scenarios"
tags:
  - glossary
aliases: []
relates_to:
  - type: "parent"
    target: "[[SYS-CLIENT-SDK]]"
  - type: "refines"
    target: "[[GLOSSARY-SUPABASE-REEF]]"
  - type: "related"
    target: "[[API-CLIENT-SDK]]"
  - type: "related"
    target: "[[DEC-CLIENT-SDK-AUTH-TOKEN-INJECTION]]"
sources:
  - category: "implementation"
    type: "code"
    ref: "packages/core/supabase-js/src/SupabaseClient.ts"
  - category: "implementation"
    type: "code"
    ref: "packages/core/supabase-js/src/lib/fetch.ts"
  - category: "implementation"
    type: "code"
    ref: "packages/core/supabase-js/src/lib/types.ts"
  - category: "implementation"
    type: "code"
    ref: "packages/core/supabase-js/src/index.ts"
  - category: "implementation"
    type: "code"
    ref: "packages/core/postgrest-js/src/types/common/common.ts"
  - category: "implementation"
    type: "code"
    ref: "packages/core/realtime-js/src/RealtimeChannel.ts"
  - category: "implementation"
    type: "code"
    ref: "packages/core/functions-js/src/FunctionsClient.ts"
  - category: "artifact"
    type: "reef"
    ref: "packages/core/supabase-js/src/lib/helpers.ts"
  - category: "artifact"
    type: "reef"
    ref: "packages/core/supabase-js/src/lib/SupabaseAuthClient.ts"
notes: ""
---

## Terms

| Term | Definition | Code Location | See Also |
|------|-----------|---------------|----------|
| SupabaseClient | Main entry-point class that composes all sub-clients (auth, database, realtime, storage, functions). Instantiated via `createClient(url, key, options)`. | `packages/core/supabase-js/src/SupabaseClient.ts` | [[SYS-CLIENT-SDK]] |
| createClient | Factory function exported from the SDK that constructs and returns a SupabaseClient instance. The recommended way to initialize the SDK. | `packages/core/supabase-js/src/index.ts` | [[API-CLIENT-SDK]] |
| Sub-client | Any of the five independent service-specific client classes (GoTrueClient, PostgrestClient, RealtimeClient, StorageClient, FunctionsClient) composed into SupabaseClient. Each is also usable as a standalone npm package. | `packages/core/supabase-js/src/SupabaseClient.ts` | [[SYS-CLIENT-SDK]] |
| GoTrueClient | The auth-js client class, named after the GoTrue authentication server it communicates with. Manages sessions, tokens, MFA, and auth state change events. Aliased as `AuthClient` for public export. | `packages/core/auth-js/src/GoTrueClient.ts`, `packages/core/auth-js/src/AuthClient.ts` | [[GLOSSARY-AUTH]] |
| SupabaseAuthClient | Thin subclass of AuthClient (GoTrueClient) used inside SupabaseClient. Exists to allow SupabaseClient-specific constructor wiring without modifying the standalone auth-js library. | `packages/core/supabase-js/src/lib/SupabaseAuthClient.ts` | |
| PostgrestClient | HTTP client that generates PostgREST-compatible REST queries from a fluent TypeScript builder API. Accessed via `supabase.from('table')` or `supabase.rpc('fn')`. | `packages/core/postgrest-js/src/PostgrestClient.ts` | |
| PostgrestQueryBuilder | Builder class returned by `.from()` that exposes `.select()`, `.insert()`, `.update()`, `.upsert()`, `.delete()` methods for constructing PostgREST HTTP requests. | `packages/core/postgrest-js/src/PostgrestQueryBuilder.ts` | [[API-CLIENT-SDK]] |
| PostgrestFilterBuilder | Builder class for adding filter predicates (`.eq()`, `.gt()`, `.in()`, etc.) to a PostgREST query. Returned after calling a query method on PostgrestQueryBuilder. | `packages/core/postgrest-js/src/PostgrestFilterBuilder.ts` | |
| RealtimeClient | WebSocket client for the Supabase Realtime server. Manages connection lifecycle, heartbeats, reconnection, and channel multiplexing. | `packages/core/realtime-js/src/RealtimeClient.ts` | [[GLOSSARY-REALTIME]] |
| RealtimeChannel | A named topic within a Realtime WebSocket connection. Supports three features: broadcast (pub/sub messages), presence (online user tracking), and postgres_changes (CDC events). Created via `supabase.channel('name')`. | `packages/core/realtime-js/src/RealtimeChannel.ts` | [[GLOSSARY-REALTIME]] |
| StorageClient | HTTP client for the Supabase Storage service. Extends StorageBucketApi (bucket-level operations) and exposes `.from('bucket')` to get a StorageFileApi for file operations. | `packages/core/storage-js/src/StorageClient.ts` | [[GLOSSARY-STORAGE]] |
| FunctionsClient | HTTP client for invoking Supabase Edge Functions. Instantiated fresh on every `supabase.functions` access (getter pattern). Supports regional routing, timeout, and abort signals. | `packages/core/functions-js/src/FunctionsClient.ts` | |
| fetchWithAuth | Higher-order function that wraps any `fetch` implementation to automatically inject `apikey` and `Authorization` headers using the current session token. The core mechanism for cross-cutting auth token injection across all HTTP sub-clients. | `packages/core/supabase-js/src/lib/fetch.ts` | [[DEC-CLIENT-SDK-AUTH-TOKEN-INJECTION]] |
| accessToken | Optional user-supplied async callback (`() => Promise<string \| null>`) for third-party auth integration (e.g., Clerk, Auth0). When set, replaces the built-in auth session as the token source and disables `supabase.auth.*` via a Proxy. | `packages/core/supabase-js/src/lib/types.ts` | [[DEC-CLIENT-SDK-AUTH-TOKEN-INJECTION]] |
| GenericSchema | TypeScript type representing a Postgres schema's structure, containing `Tables`, `Views`, and `Functions` record types. Used by the generic type system to provide compile-time type safety for database queries. | `packages/core/postgrest-js/src/types/common/common.ts` (synced to `supabase-js/src/lib/rest/types/common/common.ts`) | |
| GenericTable | TypeScript type representing a database table with `Row`, `Insert`, `Update`, and `Relationships` sub-types. Enables type-safe CRUD operations through the PostgREST query builder. | `packages/core/postgrest-js/src/types/common/common.ts` | |
| Fixed versioning | Release strategy where all six monorepo packages share identical version numbers (e.g., all at 2.80.0). Managed by Nx release with `projectsRelationship: "fixed"`. | `nx.json`, `packages/core/supabase-js/package.json` | |
| Canary release | Automated pre-release published to npm on every commit to master. Uses the `canary` dist-tag (e.g., `2.80.1-canary.0`). Allows early testing before manual promotion to stable `latest`. | `.github/workflows/main-ci-release.yml` | |
| Wildcard dependency (`*`) | Internal workspace dependencies use `"*"` as version specifier in package.json. Nx replaces these with the actual version number during the release process. | `packages/core/supabase-js/package.json` | |
| X-Client-Info | HTTP header automatically set on all requests, identifying the SDK runtime environment and version (e.g., `supabase-js-web/2.80.0`, `supabase-js-node/2.80.0`). Detected at module load time. | `packages/core/supabase-js/src/lib/constants.ts` | |
| FunctionRegion | Enum specifying the AWS region where an Edge Function should execute. Passed as the `x-region` header and `forceFunctionRegion` query parameter. Defaults to `FunctionRegion.Any`. | `packages/core/functions-js/src/types.ts` | |
| onAuthStateChange | Method on the auth client that registers a callback for auth events (`SIGNED_IN`, `SIGNED_OUT`, `TOKEN_REFRESHED`, etc.). SupabaseClient uses it internally to push token updates to the Realtime sub-client. | `packages/core/auth-js/src/GoTrueClient.ts`, `packages/core/supabase-js/src/SupabaseClient.ts` | [[PROC-CLIENT-SDK-AUTH]] |
| flowType | Auth configuration option selecting the OAuth flow: `implicit` (default, token in URL fragment) or `pkce` (recommended for mobile/server-side, uses code verifier). | `packages/core/supabase-js/src/lib/constants.ts`, `packages/core/supabase-js/src/lib/types.ts` | [[GLOSSARY-AUTH]] |
| storageKey | String key used to persist the auth session in local/async storage. Defaults to `sb-{project-ref}-auth-token`, namespaced by the project's hostname. | `packages/core/supabase-js/src/SupabaseClient.ts` line 294 | |

## Naming Conventions

- **Package names**: `@supabase/{name}-js` on npm (e.g., `@supabase/auth-js`, `@supabase/postgrest-js`)
- **Source location**: `packages/core/{name}/src/` in the monorepo
- **Main client classes**: PascalCase with `Client` suffix (SupabaseClient, GoTrueClient, RealtimeClient, PostgrestClient, StorageClient, FunctionsClient)
- **Builder classes**: PascalCase with `Builder` suffix (PostgrestQueryBuilder, PostgrestFilterBuilder, PostgrestTransformBuilder)
- **Error classes**: PascalCase describing the error domain (FunctionsHttpError, FunctionsRelayError, FunctionsFetchError, PostgrestError, AuthError)
- **Type helpers**: PascalCase exported types for query result inference (QueryResult, QueryData, QueryError, DatabaseWithoutInternals)

## Related

- [[SYS-CLIENT-SDK]] -- parent system
- [[GLOSSARY-SUPABASE-REEF]] -- unified cross-service glossary
- [[API-CLIENT-SDK]] -- API surface using these terms
- [[DEC-CLIENT-SDK-AUTH-TOKEN-INJECTION]] -- decision on auth injection pattern
