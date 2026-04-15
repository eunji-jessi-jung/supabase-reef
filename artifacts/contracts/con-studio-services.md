---
id: "CON-STUDIO-SERVICES"
type: "contract"
title: "Studio to Backend Services Integration"
domain: "supabase-reef"
status: "draft"
last_verified: 2026-04-15
freshness_note: "Enriched with Studio data layer analysis: auth, storage, realtime hooks and fetcher patterns traced from source"
freshness_triggers:
  - "apps/studio/data/"
  - "packages/api-types/types/"
known_unknowns:
  - "How Studio obtains service_role credentials for backend calls"
  - "Which specific operations go through BFF routes vs direct platform API calls"
tags:
  - contract
  - rest-api
aliases: []
relates_to:
  - type: "integrates_with"
    target: "[[SYS-STUDIO]]"
  - type: "integrates_with"
    target: "[[SYS-AUTH]]"
  - type: "integrates_with"
    target: "[[SYS-STORAGE]]"
  - type: "integrates_with"
    target: "[[SYS-REALTIME]]"
sources:
  - category: "implementation"
    type: "github"
    ref: "apps/studio/data/__templates/keys.ts"
  - category: "implementation"
    type: "github"
    ref: "packages/api-types/types/api.d.ts"
  - category: "implementation"
    type: "github"
    ref: "apps/studio/data/fetchers.ts"
    notes: "Central HTTP client with auth middleware"
  - category: "implementation"
    type: "github"
    ref: "apps/studio/data/auth/"
    notes: "Auth service integration hooks"
  - category: "implementation"
    type: "github"
    ref: "apps/studio/data/storage/"
    notes: "Storage service integration hooks"
  - category: "implementation"
    type: "github"
    ref: "apps/studio/data/realtime/"
    notes: "Realtime service integration hooks"
notes: ""
---

## Parties

- **Consumer**: Studio (dashboard UI that manages all Supabase services)
- **Producers**: Auth, Storage, Realtime, PostgREST (backend services)

## Key Facts

- Studio data layer has 60+ domain-specific modules under `apps/studio/data/`, each containing React Query hooks and fetcher functions for a specific backend concern -> `apps/studio/data/`
- api-types package provides TypeScript interfaces for both the v1 Management API (163 operations) and Platform API (345 operations), totaling 508 operations across 360 paths -> `packages/api-types/types/`
- Studio BFF routes in `apps/studio/app/api/` proxy to backend services -> `apps/studio/app/api/`
- pg-meta provides Postgres introspection used by the database management features -> `packages/pg-meta/`
- All HTTP calls go through `openapi-fetch` client parameterized with api-types `paths`, providing compile-time type safety on every request -> `apps/studio/data/fetchers.ts`
- Auth middleware injects `Authorization: Bearer` token and `X-Request-Id` on every request via `constructHeaders()` -> `apps/studio/data/fetchers.ts`

### Auth Service Integration

- Studio manages Auth configuration (providers, hooks, email templates) via `/platform/auth/{ref}/config` endpoints -> `apps/studio/data/auth/auth-config-query.ts`
- User management (CRUD, password reset, magic link, OTP, MFA factor deletion) via `/platform/auth/{ref}/users` -> `apps/studio/data/auth/`
- Custom OAuth provider management via dedicated data module -> `apps/studio/data/oauth-custom-providers/`
- SSO provider management via dedicated data module -> `apps/studio/data/sso/`
- Third-party auth configuration -> `apps/studio/data/third-party-auth/`

### Storage Service Integration

- Bucket CRUD (create, update, delete, empty) via `/platform/storage/{ref}/buckets` -> `apps/studio/data/storage/`
- Object operations (list, delete, move, sign URLs, get public URLs) via `/platform/storage/{ref}/objects` -> `apps/studio/data/storage/`
- S3 access key management (create, delete, query) -> `apps/studio/data/storage/s3-access-key-*.ts`
- Iceberg namespace and table management -> `apps/studio/data/storage/iceberg-*.ts`
- Vector bucket and index management -> `apps/studio/data/storage/vector-*.ts`
- Storage archive operations -> `apps/studio/data/storage/storage-archive-*.ts`

### Realtime Service Integration

- Realtime configuration (rate limits, connection pool, max users) via `/platform/projects/{ref}/config/realtime` with fallback to default config values -> `apps/studio/data/realtime/realtime-config-query.ts`
- Default config: `max_concurrent_users: 200`, `max_events_per_second: 100`, `max_bytes_per_second: 100000`, `max_channels_per_client: 100` -> `apps/studio/data/realtime/realtime-config-query.ts`
- Configuration mutation (update rate limits) via same endpoint -> `apps/studio/data/realtime/realtime-config-mutation.ts`

### Cross-Service Patterns

- All service interactions are gated by `IS_PLATFORM` flag; self-hosted Studio may bypass some service calls -> `apps/studio/lib/constants/index.ts`
- pg-meta requests require `x-connection-encrypted` header, enforced by `pgMetaGuard` middleware -> `apps/studio/data/fetchers.ts`
- Service endpoints follow consistent path patterns: `/platform/{service}/{ref}/...` for platform API, `/v1/projects/{ref}/...` for management API

## Agreement

Studio interacts with all backend services through the Supabase Platform API, using api-types as the typed contract. The platform API acts as a gateway that routes to individual service APIs (Auth/GoTrue, Storage, Realtime). Studio uses session-based auth tokens (from its own GoTrue login) for all platform API calls. The `credentials: 'include'` setting enables cookie-based session transport alongside Bearer tokens.

## Current State

The contract is semi-formal through TypeScript types in api-types. The types are regenerated via `openapi-typescript` from live platform API specs. Studio has comprehensive data hooks covering auth management, storage management, and realtime configuration. The BFF layer (`apps/studio/app/api/`) proxies some requests, particularly for operations requiring server-side secrets.

## Related

- [[SYS-STUDIO]] -- consumer
- [[SYS-AUTH]] -- producer
- [[SYS-STORAGE]] -- producer
- [[SYS-REALTIME]] -- producer
