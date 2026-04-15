---
id: "DEC-STORAGE-RLS-AUTH-DELEGATION"
type: "decision"
title: "Storage Delegates Authorization to Postgres Row Level Security"
domain: "storage"
status: "draft"
last_verified: 2026-04-15
freshness_note: "Deep read of connection.ts, db.ts, jwt.ts, knex.ts, storage.ts, object.ts, uploader.ts"
freshness_triggers:
  - "src/internal/database/connection.ts"
  - "src/http/plugins/db.ts"
  - "src/http/plugins/jwt.ts"
  - "src/storage/database/knex.ts"
  - "src/storage/storage.ts"
known_unknowns:
  - "Exact RLS policy definitions per table — policies live in tenant databases, not in this codebase"
  - "Performance impact of per-transaction SET LOCAL calls versus application-level permission checks"
  - "Whether any protocol bypasses RLS evaluation (vector routes enforce service_role only, skipping RLS)"
tags:
  - decision
  - architecture
  - rls
  - auth
  - postgres
aliases: []
relates_to:
  - type: "parent"
    target: "[[SYS-STORAGE]]"
  - type: "related"
    target: "[[DEC-MULTI-PROTOCOL-STORAGE]]"
  - type: "related"
    target: "[[PROC-STORAGE-AUTH]]"
  - type: "related"
    target: "[[CON-AUTH-STORAGE]]"
sources:
  - category: "implementation"
    type: "code"
    ref: "src/internal/database/connection.ts"
    notes: "TenantConnection.setScope() — sets 10 GUC variables per transaction including role, JWT claims, request context"
  - category: "implementation"
    type: "code"
    ref: "src/http/plugins/db.ts"
    notes: "db Fastify plugin — creates per-request TenantConnection with user JWT payload and admin superUser"
  - category: "implementation"
    type: "code"
    ref: "src/http/plugins/jwt.ts"
    notes: "JWT verification plugin — extracts role from JWT, defaults to 'anon' when missing or invalid"
  - category: "implementation"
    type: "code"
    ref: "src/storage/database/knex.ts"
    notes: "StorageKnexDB — asSuperUser(), testPermission(), withTransaction() with setScope() call"
  - category: "implementation"
    type: "code"
    ref: "src/storage/storage.ts"
    notes: "Storage.asSuperUser() — creates new Storage instance with superuser DB connection"
  - category: "implementation"
    type: "code"
    ref: "src/storage/uploader.ts"
    notes: "Uploader.canUpload() uses testPermission() to check RLS without side effects"
  - category: "implementation"
    type: "code"
    ref: "src/config.ts"
    notes: "Configurable role names: dbAnonRole, dbServiceRole, dbAuthenticatedRole"
  - category: "context"
    type: "documentation"
    ref: "sources/context/decisions/architecture-overview.md"
    notes: "Supabase product principles: 'everything works in isolation', 'everything is extensible'"
notes: ""
---

## Context

Storage needs to enforce per-user and per-bucket access control across four protocols (HTTP, TUS, S3, Iceberg) and multiple operations (upload, download, delete, list, copy, move). Rather than implementing a custom RBAC/ABAC system in the application layer, the service must decide where authorization logic lives and how it is enforced.

## Decision

Delegate all object and bucket authorization to Postgres Row Level Security (RLS) policies by injecting the caller's JWT claims and role into the database session before every transaction. Storage never checks permissions in application code — it relies on Postgres to accept or reject queries based on RLS policies defined in the tenant's database.

## Key Facts

- Every transaction begins with `TenantConnection.setScope()`, which executes a single SQL statement setting 10 GUC (Grand Unified Configuration) variables: `role`, `request.jwt.claim.role`, `request.jwt`, `request.jwt.claim.sub`, `request.jwt.claims`, `request.headers`, `request.method`, `request.path`, `storage.operation`, and `storage.allow_delete_query` -> `src/internal/database/connection.ts:170-197`
- The role is extracted from the JWT payload at connection creation time, defaulting to `'anon'` if absent -> `src/internal/database/connection.ts:31`
- Three database roles are configurable via env vars: `DB_ANON_ROLE` (default `'anon'`), `DB_SERVICE_ROLE` (default `'service_role'`), `DB_AUTHENTICATED_ROLE` (default `'authenticated'`) -> `src/config.ts:387-389`
- The `db` Fastify plugin creates a per-request `TenantConnection` using the verified JWT payload as the user and a service-key-signed admin user as the superUser -> `src/http/plugins/db.ts:37-57`
- `StorageKnexDB.asSuperUser()` returns a new DB instance that uses the `superUser` credentials, bypassing RLS for internal operations like background deletes -> `src/storage/database/knex.ts:97-103`
- `testPermission()` wraps a query in a transaction that always rolls back (`TestPermissionRollbackError`), allowing the service to check whether RLS would permit an operation without side effects -> `src/storage/database/knex.ts:105-118`
- The Uploader's `canUpload()` method uses `testPermission()` to pre-check whether the caller can create an object before streaming bytes to the backend, avoiding orphaned S3 uploads -> `src/storage/uploader.ts:70-80`
- `Storage.asSuperUser()` creates a new `Storage` instance wrapping a superuser DB, used when internal operations (e.g., bucket deletion, background jobs) must bypass RLS -> `src/storage/storage.ts:50-52`
- When JWT verification fails and the route allows invalid JWTs (`allowInvalidJwt`), the payload defaults to `{ role: 'anon' }` and `isAuthenticated` is set to false, so RLS anon policies still apply -> `src/http/plugins/jwt.ts:43-46, 58-59`
- The `dbSuperUser` plugin variant always uses admin credentials for both user and superUser, used by admin routes that intentionally bypass RLS -> `src/http/plugins/db.ts:107-133`
- Vector routes enforce `service_role` JWT via `enforceJwtRoles`, meaning they skip per-user RLS and rely on role-level access instead -> `src/http/routes/vector/index.ts:50`
- All `set_config` calls use the third parameter `true` (local to transaction), so GUC variables are scoped to the current transaction only and do not leak across connections -> `src/internal/database/connection.ts:172-197`

## Rationale

Not fully documented in code. The design aligns with Supabase's platform-wide pattern where Postgres RLS is the single source of truth for authorization across all services (PostgREST, Realtime, Storage). Benefits observed from implementation:

1. **Policy-as-data**: Authorization rules are SQL expressions in the tenant database, editable by users without redeploying Storage.
2. **Single enforcement point**: All four protocols share the same `setScope()` -> RLS evaluation path, eliminating the risk of protocol-specific permission bypasses.
3. **Consistency with PostgREST**: Uses the same GUC variable names (`request.jwt.claims`, `request.jwt.claim.sub`, `request.jwt.claim.role`) as PostgREST, so users write one set of policies for both APIs.
4. **Everything works in isolation**: Storage works with just Postgres + RLS for authorization — no separate auth service is required to enforce access control. A user can run Storage with nothing but a Postgres database, satisfying Supabase's litmus test for isolation. -> `sources/context/decisions/architecture-overview.md`
5. **Everything is extensible**: Users write standard Postgres RLS policies (SQL expressions) rather than custom auth rules in a proprietary format. This leverages an existing Postgres primitive instead of adding a new authorization system. -> `sources/context/decisions/architecture-overview.md`

## Consequences

### Positive
- Authorization logic is fully customizable by end users via standard SQL policies
- Adding a new protocol or route automatically inherits existing RLS rules with zero extra permission code
- `testPermission()` enables pre-flight permission checks without side effects, preventing wasted backend writes
- SuperUser escape hatch (`asSuperUser()`) allows internal operations to bypass RLS when needed

### Negative
- Every transaction pays the cost of 10 `set_config()` calls before any business query
- Debugging authorization failures requires understanding both application-level JWT extraction and database-level RLS policies
- RLS policy errors surface as opaque Postgres errors that must be translated into meaningful HTTP responses
- Storage cannot enforce fine-grained permissions beyond what SQL expressions can evaluate (no external policy engines)

### Neutral
- Storage owns no permission logic of its own — it is fully dependent on the tenant database having correct RLS policies installed
- The `storage.allow_delete_query` GUC acts as a soft-delete protection mechanism rather than a user-facing permission

## Related

- [[SYS-STORAGE]] -- parent system
- [[DEC-MULTI-PROTOCOL-STORAGE]] -- all protocols share this auth path
- [[PROC-STORAGE-AUTH]] -- detailed auth flow process
- [[CON-AUTH-STORAGE]] -- auth integration contract
