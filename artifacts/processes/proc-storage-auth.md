---
id: "PROC-STORAGE-AUTH"
type: "process"
title: "Storage Authentication and Authorization Flow"
domain: "supabase-reef"
status: draft
last_verified: 2026-04-15
freshness_note: "Deep read of auth plugins, JWT verification, RLS scope-setting, and S3 signature-v4 flow."
freshness_triggers:
  - "src/http/plugins/apikey.ts"
  - "src/http/plugins/db.ts"
  - "src/http/plugins/jwt.ts"
  - "src/http/plugins/signature-v4.ts"
  - "src/internal/auth/jwt.ts"
  - "src/internal/database/connection.ts"
  - "src/internal/database/tenant.ts"
known_unknowns:
  - "Exact Postgres RLS policy definitions are in SQL migrations, not reviewed here"
  - "How JWKS key rotation propagates to running instances beyond PubSub invalidation"
tags:
  - storage
  - auth
  - jwt
  - rls
  - s3
  - signature-v4
aliases:
  - "Storage Auth Flow"
relates_to:
  - type: "depends_on"
    target: "[[CON-AUTH-STORAGE]]"
  - type: "parent"
    target: "[[SYS-STORAGE]]"
  - type: "refines"
    target: "[[DEC-MULTI-PROTOCOL-STORAGE]]"
sources:
  - category: implementation
    type: github
    ref: "src/http/plugins/apikey.ts"
    notes: "Admin API key authentication plugin"
  - category: implementation
    type: github
    ref: "src/http/plugins/db.ts"
    notes: "DB connection plugin sets JWT claims on Postgres connection"
  - category: implementation
    type: github
    ref: "src/http/plugins/jwt.ts"
    notes: "Fastify JWT preHandler plugin"
  - category: implementation
    type: github
    ref: "src/http/plugins/signature-v4.ts"
    notes: "AWS Signature V4 auth for S3 protocol"
  - category: implementation
    type: github
    ref: "src/internal/auth/jwt.ts"
    notes: "Core JWT verification, caching, signing, JWKS support"
  - category: implementation
    type: github
    ref: "src/internal/database/connection.ts"
    notes: "TenantConnection.setScope sets RLS context per-transaction"
  - category: implementation
    type: github
    ref: "src/internal/database/tenant.ts"
    notes: "Tenant config resolution, JWT secret retrieval, JWKS merging"
notes: ""
---

## Purpose

Documents how Storage authenticates and authorizes every request across its three protocols (REST/HTTP, S3, TUS). Storage uses JWTs issued by Supabase Auth and verified locally, then configures each Postgres connection with JWT claims so that RLS policies on `storage.buckets` and `storage.objects` govern access. The S3 protocol adds AWS Signature V4 as an alternative authentication path that ultimately produces the same JWT-based Postgres session.

## Key Facts

- JWT plugin extracts the Bearer token from the Authorization header, strips the `Bearer ` prefix via regex, and stores it on `request.jwt` → `src/http/plugins/jwt.ts`
- When no JWT is present and `allowInvalidJwt` is set on the route config, the request proceeds as unauthenticated with `role: 'anon'` and `isAuthenticated: false` → `src/http/plugins/jwt.ts`
- JWT verification supports HMAC (HS256/384/512), RSA (RS256/384/512), ECC (ES256/384/512), and EdDSA algorithms, dynamically selecting allowed algorithms from the JWKS key set → `src/internal/auth/jwt.ts`
- Verified JWT payloads are cached in an LRU cache (max 65536 items, 50 MiB, TTL matched to token expiry) keyed by SHA-256 of token+secret+JWKS fingerprint → `src/internal/auth/jwt.ts`
- The `enforceJwtRole` plugin adds a second preHandler that rejects requests whose JWT role is not in the allowed list, returning 403 → `src/http/plugins/jwt.ts`
- `TenantConnection.setScope()` calls `set_config` on the Postgres transaction to set `role`, `request.jwt.claim.role`, `request.jwt`, `request.jwt.claim.sub`, `request.jwt.claims`, `request.headers`, `request.method`, `request.path`, and `storage.operation` -- these are the values RLS policies evaluate → `src/internal/database/connection.ts`
- The DB plugin (`db-init`) creates a Postgres connection using both the user JWT payload and a service-key super-user; the super-user is fetched via `getServiceKeyUser()` which decrypts the tenant's stored service key → `src/http/plugins/db.ts`
- In multi-tenant mode, JWT secrets are per-tenant: `getJwtSecret()` decrypts the tenant's `jwt_secret` and merges tenant-level JWKS keys with legacy JWKS keys from the tenants table → `src/internal/database/tenant.ts`
- S3 Signature V4 authentication extracts credentials from the Authorization header, query string (`X-Amz-Credential`), or multipart form data, then verifies the signature server-side → `src/http/plugins/signature-v4.ts`
- When an S3 request uses a session token, the token is treated as a JWT and verified against the tenant's JWT secret; otherwise the S3 access key maps to stored credentials with embedded claims → `src/http/plugins/signature-v4.ts`
- After successful S3 signature verification, the plugin synthesizes a short-lived (5m) JWT from the credential claims or verifies the session-token JWT, setting `request.jwt`, `request.jwtPayload`, and `request.owner` identically to the JWT plugin → `src/http/plugins/signature-v4.ts`
- Admin API key authentication (`auth-admin-api-key` plugin) validates a comma-separated set of admin keys from config against the `apikey` header on admin routes → `src/http/plugins/apikey.ts`
- Service role / super-user connections use `dbSuperUser` plugin which bypasses per-user JWT claim scoping, effectively bypassing RLS for admin operations → `src/http/plugins/db.ts`
- DB connections are disposed on `onSend`, `onTimeout`, and `onRequestAbort` hooks, ensuring connection cleanup even on failed or aborted requests → `src/http/plugins/db.ts`

## Steps

1. **Tenant resolution** -- `tenant-id` plugin extracts tenant ID from `x-forwarded-host` header (multi-tenant) or uses the configured default (single-tenant).
2. **Authentication** -- Depending on protocol:
   - **REST/HTTP**: `jwt` preHandler extracts and verifies the Bearer JWT via `verifyJWT` or `verifyJWTWithCache`.
   - **S3**: `signature-v4` plugin verifies AWS Signature V4 using stored S3 credentials or session-token JWT, then synthesizes/verifies a JWT for downstream use.
   - **TUS**: Uses the same JWT plugin; signed-upload variant verifies an `x-signature` header against a pre-signed upload token.
   - **Admin**: `auth-admin-api-key` plugin checks the `apikey` header against configured admin keys.
3. **Role enforcement** -- Optional `enforceJwtRole` preHandler validates the JWT role against a whitelist (e.g., `service_role` only).
4. **Database connection** -- `db-init` preHandler creates a `TenantConnection` with the user's JWT payload and a super-user fallback.
5. **RLS scope injection** -- `TenantConnection.setScope()` runs `SELECT set_config(...)` to inject 10 session variables (role, claims, headers, path, method, operation) into the Postgres transaction.
6. **Query execution** -- All storage queries run within transactions where RLS policies on `storage.buckets` and `storage.objects` filter results based on the injected claims.
7. **Backend access** -- Object reads/writes to S3 backend use internal service credentials, not the user's JWT.
8. **Cleanup** -- DB connections are disposed via `onSend`/`onTimeout`/`onRequestAbort` hooks.

## Related

- [[SYS-STORAGE]] -- parent system
- [[CON-AUTH-STORAGE]] -- cross-service contract defining JWT expectations
- [[DEC-MULTI-PROTOCOL-STORAGE]] -- decision record for multi-protocol support (REST, S3, TUS)
