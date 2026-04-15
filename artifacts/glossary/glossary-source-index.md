---
id: "GLOSSARY-SOURCE-INDEX"
type: "glossary"
title: "Cross-Service Source Index"
domain: "supabase-reef"
status: draft
last_verified: 2026-04-15
freshness_note: "scuba-depth cross-service term-to-source mapping"
freshness_triggers:
  - "internal/models/"
  - "lib/realtime/"
  - "src/storage/"
  - "apps/studio/"
  - "packages/"
known_unknowns:
  - "PostgREST code locations not mapped (external service, not in Supabase monorepo source)"
  - "Edge Functions runtime code locations not yet traced"
  - "Studio component paths for SQL Editor and Table Editor need deeper tracing to exact files"
tags: ["glossary", "cross-service", "source-index"]
aliases: []
relates_to:
  - type: "refines"
    target: "[[GLOSSARY-SUPABASE-REEF]]"
  - type: "refines"
    target: "[[GLOSSARY-AUTH]]"
  - type: "refines"
    target: "[[GLOSSARY-REALTIME]]"
  - type: "refines"
    target: "[[GLOSSARY-STORAGE]]"
  - type: "refines"
    target: "[[GLOSSARY-STUDIO]]"
  - type: "refines"
    target: "[[GLOSSARY-CLIENT-SDK]]"
sources:
  - category: "implementation"
    type: "code"
    ref: "internal/api/auth.go"
  - category: "implementation"
    type: "code"
    ref: "lib/realtime.ex"
  - category: "implementation"
    type: "code"
    ref: "src/index.ts"
  - category: "implementation"
    type: "code"
    ref: "apps/studio/package.json"
  - category: "implementation"
    type: "code"
    ref: "packages/core/supabase-js/src/SupabaseClient.ts"
notes: "This index is synthesized from all per-service glossaries. Each cell contains the primary code path where the term is defined or implemented in that service. A dash (—) means the term is not used in that service."
---

## How to Read This Index

Each cell contains the **primary code path** where a term is defined or implemented within that service's codebase. A dash (**—**) means the term does not appear or is not relevant in that service. Paths are relative to each service's repository root.

## Cross-Service Terms

These terms appear in two or more services and are the most important for cross-team communication.

| Term | Auth | Realtime | Storage | Studio | Client SDK |
|------|------|----------|---------|--------|------------|
| JWT | `internal/api/auth.go` | validated on WebSocket join in `lib/realtime_web/channels/` | validated on HTTP requests in `src/http/routes/` | consumed in `apps/studio/data/` | injected via `packages/core/supabase-js/src/lib/fetch.ts` |
| RLS | `auth` schema tables | `lib/extensions/` (postgres_cdc_rls) | `src/http/routes/bucket/`, `src/http/routes/object/` | `apps/studio/data/database-policies/` | — |
| service_role | `internal/api/auth.go` | tenant config in `lib/realtime/api/tenant.ex` | used in admin routes in `src/http/routes/` | `apps/studio/data/` | — |
| Project / Tenant | implicit per-config | `lib/realtime/api/tenant.ex` | implicit via tenant_id | `apps/studio/data/` | `packages/core/supabase-js/src/SupabaseClient.ts` |
| PKCE | `internal/api/token.go` | — | — | — | `packages/core/auth-js/src/GoTrueClient.ts` |
| GoTrue | `cmd/root_cmd.go` | — | — | — | `packages/core/auth-js/src/GoTrueClient.ts` |
| Channel | — | `lib/realtime_web/channels/realtime_channel.ex` | — | — | `packages/core/realtime-js/src/` |
| PostgREST | — | — | — | `packages/pg-meta/src/` | `packages/core/postgrest-js/src/` |

## Auth-Specific Terms

| Term | Auth | Realtime | Storage | Studio | Client SDK |
|------|------|----------|---------|--------|------------|
| AAL | `internal/models/sessions.go` | — | — | — | `packages/core/auth-js/src/` |
| Factor (MFA) | `internal/models/factor.go` | — | — | — | `packages/core/auth-js/src/` |
| Identity | `internal/models/identity.go` | — | — | — | — |
| FlowState | `internal/models/flow_state.go` | — | — | — | — |
| Hooks | `internal/hooks/` | — | — | — | — |
| OAuthClient | `internal/models/oauth_client.go` | — | — | — | — |
| AMR | `internal/models/amr.go` | — | — | — | — |
| WebAuthn | `internal/models/webauthn_credential.go` | — | — | — | — |
| Captcha | `internal/security/captcha.go` | — | — | — | — |
| HIBP | `internal/api/api.go` | — | — | — | — |

## Realtime-Specific Terms

| Term | Auth | Realtime | Storage | Studio | Client SDK |
|------|------|----------|---------|--------|------------|
| Broadcast | — | `lib/realtime_web/channels/realtime_channel.ex` | — | — | `packages/core/realtime-js/src/` |
| Presence | — | `lib/realtime_web/channels/realtime_channel.ex` | — | — | `packages/core/realtime-js/src/` |
| Postgres Changes | — | `lib/realtime/postgres_cdc.ex` | — | — | `packages/core/realtime-js/src/` |
| Extension | — | `lib/extensions/` | — | — | — |
| Replication Connection | — | `lib/realtime/tenants/replication_connection.ex` | — | — | — |
| Janitor | — | `lib/realtime/tenants/janitor.ex` | — | — | — |
| Rebalancer | — | `lib/realtime/tenants/rebalancer.ex` | — | — | — |
| GenCounter | — | `lib/realtime/gen_counter/` | — | — | — |
| Syn | — | `lib/realtime/syn/` | — | — | — |

## Storage-Specific Terms

| Term | Auth | Realtime | Storage | Studio | Client SDK |
|------|------|----------|---------|--------|------------|
| Bucket | — | — | `src/http/routes/bucket/` | — | `packages/core/storage-js/src/` |
| Object | — | — | `src/storage/object.ts` | — | `packages/core/storage-js/src/` |
| Backend | — | — | `src/storage/backend/` | — | — |
| TUS | — | — | `src/storage/protocols/tus/` | — | — |
| Signed URL | — | — | `src/http/routes/object/getSignedUploadURL.ts` | — | `packages/core/storage-js/src/` |
| Render | — | — | `src/http/routes/render/` | — | — |
| Uploader | — | — | `src/storage/uploader.ts` | — | — |
| Locator | — | — | `src/storage/locator.ts` | — | — |
| Scanner | — | — | `src/storage/scanner/` | — | — |
| Iceberg | — | — | `src/http/routes/iceberg/` | — | — |
| Multipart Upload | — | — | `src/storage/protocols/s3/s3-handler.ts` | — | — |

## Studio-Specific Terms

| Term | Auth | Realtime | Storage | Studio | Client SDK |
|------|------|----------|---------|--------|------------|
| pg-meta | — | — | — | `packages/pg-meta/src/` | — |
| api-types | — | — | — | `packages/api-types/types/` | — |
| Branch | — | — | — | `apps/studio/data/branches/` | — |
| Data hooks | — | — | — | `apps/studio/data/` | — |
| BFF | — | — | — | `apps/studio/app/api/` | — |
| SQL Editor | — | — | — | `apps/studio/components/` | — |
| Table Editor | — | — | — | `apps/studio/components/` | — |

## Client SDK-Specific Terms

| Term | Auth | Realtime | Storage | Studio | Client SDK |
|------|------|----------|---------|--------|------------|
| SupabaseClient | — | — | — | — | `packages/core/supabase-js/src/SupabaseClient.ts` |
| GoTrueClient | — | — | — | — | `packages/core/auth-js/src/GoTrueClient.ts` |
| RealtimeClient | — | — | — | — | `packages/core/realtime-js/src/` |
| StorageClient | — | — | — | — | `packages/core/storage-js/src/` |
| FunctionsClient | — | — | — | — | `packages/core/functions-js/src/` |
| fetchWithAuth | — | — | — | — | `packages/core/supabase-js/src/lib/fetch.ts` |
| Canary release | — | — | — | — | `.github/workflows/` |
| Fixed versioning | — | — | — | — | `nx.json` |
| Edge Functions | — | — | — | — | `packages/core/functions-js/src/` |

## Key Facts

1. **JWT is the universal currency** — issued by Auth (`internal/api/auth.go`), validated independently by Realtime (WebSocket join), Storage (HTTP middleware), and injected by the Client SDK (`fetchWithAuth`). Source: [[GLOSSARY-SUPABASE-REEF]], [[GLOSSARY-AUTH]], [[GLOSSARY-CLIENT-SDK]].
2. **RLS spans four services** — Auth protects its own schema tables, Storage filters bucket/object access, Realtime filters CDC via `postgres_cdc_rls` extension, and Studio provides a visual policy editor at `apps/studio/data/database-policies/`. Source: [[GLOSSARY-SUPABASE-REEF]].
3. **"Project" vs "Tenant" naming split** — Realtime is the only service that uses "Tenant" (`lib/realtime/api/tenant.ex`); all other services use "Project." Source: [[GLOSSARY-SUPABASE-REEF]], [[GLOSSARY-REALTIME]].
4. **GoTrue legacy naming persists in two codebases** — Auth still uses it in `cmd/root_cmd.go` and config env vars; the Client SDK preserves it as the `GoTrueClient` class name in `packages/core/auth-js/src/GoTrueClient.ts`. Source: [[GLOSSARY-AUTH]], [[GLOSSARY-CLIENT-SDK]].
5. **Client SDK mirrors server services 1:1** — each server service (Auth, Realtime, Storage, Functions) has a dedicated `-js` client package under `packages/core/` with a matching `*Client` class. Source: [[GLOSSARY-CLIENT-SDK]].
6. **PKCE bridges Auth and Client SDK** — implemented server-side in `internal/api/token.go` and client-side in `GoTrueClient`. Not used by Realtime, Storage, or Studio. Source: [[GLOSSARY-AUTH]], [[GLOSSARY-CLIENT-SDK]].
7. **Storage has the richest protocol diversity** — TUS (resumable uploads), S3 (multipart), Iceberg (data lake), and signed URLs are all Storage-only concepts with no cross-service equivalents. Source: [[GLOSSARY-STORAGE]].
8. **Studio's pg-meta and api-types packages are shared infrastructure** — used by Studio but published as `@supabase/*` npm packages, making them available to the broader ecosystem. Source: [[GLOSSARY-STUDIO]].

## Related

- [[GLOSSARY-SUPABASE-REEF]] -- unified platform glossary with disambiguation
- [[GLOSSARY-AUTH]] -- Auth-specific terms and definitions
- [[GLOSSARY-REALTIME]] -- Realtime-specific terms and definitions
- [[GLOSSARY-STORAGE]] -- Storage-specific terms and definitions
- [[GLOSSARY-STUDIO]] -- Studio-specific terms and definitions
- [[GLOSSARY-CLIENT-SDK]] -- Client SDK-specific terms and definitions
