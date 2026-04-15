---
id: "API-STUDIO-API-TYPES"
type: "api"
title: "Studio API Types Package"
domain: "supabase-reef"
status: draft
last_verified: 2026-04-15
freshness_note: "Based on current api-types package; generated types change when upstream Management API evolves."
freshness_triggers:
  - "packages/api-types/index.ts"
  - "packages/api-types/package.json"
  - "packages/api-types/redocly.yaml"
  - "packages/api-types/types/api.d.ts"
  - "packages/api-types/types/platform.d.ts"
known_unknowns:
  - "Exact cadence of upstream OpenAPI spec updates is unknown; codegen is triggered manually."
  - "Whether api.d.ts and platform.d.ts will merge into a single spec is not documented."
tags:
  - "api-types"
  - "codegen"
  - "openapi"
  - "studio"
  - "typescript"
aliases:
  - "api-types"
  - "Management API types"
relates_to:
  - type: "parent"
    target: "[[API-STUDIO]]"
  - type: "feeds"
    target: "[[CON-STUDIO-SERVICES]]"
  - type: "feeds"
    target: "[[SYS-STUDIO]]"
sources:
  - category: implementation
    type: github
    ref: "packages/api-types/index.ts"
    notes: "Barrel export merging api and platform types into unified paths/operations/components interfaces"
  - category: implementation
    type: github
    ref: "packages/api-types/package.json"
    notes: "Codegen script using openapi-typescript ^7.4.3"
  - category: implementation
    type: github
    ref: "packages/api-types/redocly.yaml"
    notes: "Redocly config pointing codegen at two upstream API endpoints"
  - category: implementation
    type: github
    ref: "packages/api-types/types/api.d.ts"
    notes: "Auto-generated v1 Management API types (12,987 lines)"
  - category: implementation
    type: github
    ref: "packages/api-types/types/platform.d.ts"
    notes: "Auto-generated platform API types (26,811 lines)"
  - category: implementation
    type: github
    ref: "apps/studio/data/fetchers.ts"
    notes: "openapi-fetch client parameterized with api-types paths"
  - category: implementation
    type: github
    ref: "apps/studio/data/edge-functions/edge-functions-query.ts"
    notes: "Example of Studio query hook consuming api-types components"
  - category: documentation
    type: manual
    ref: "sources/apis/studio/api-types/openapi.json"
    notes: "Reef-extracted summary of operation and path counts"
notes: "The api-types package does not define an API; it provides generated TypeScript types for the Supabase Management API consumed by Studio."
---

## Overview

The `api-types` package (`packages/api-types/`) is a shared TypeScript type package in the Supabase monorepo. It contains auto-generated type definitions from the Supabase Management API's OpenAPI specifications. Studio (and other packages) import these types to get compile-time safety on every HTTP call to the platform backend.

The package does not implement any API endpoints itself. It is a **type-only dependency** that translates two upstream OpenAPI specs (the public v1 Management API and the internal platform API) into TypeScript interfaces via `openapi-typescript`.

## Key Facts

- The codegen pipeline uses `openapi-typescript ^7.4.3` with a Redocly config that fetches specs from two local endpoints: `/api/v1-json` (v1 API) and `/api/platform-json` (platform API) → `packages/api-types/redocly.yaml`
- The v1 Management API surface contains 107 paths and 163 operations, covering branches, organizations, projects, OAuth, snippets, and profile → `sources/apis/studio/api-types/openapi.json`
- The platform API surface contains 253 paths and 345 operations, covering auth, database, integrations, storage, replication, billing, and more → `sources/apis/studio/api-types/openapi.json`
- Combined, the two specs produce 508 total operations across 360 paths → `sources/apis/studio/api-types/openapi.json`
- `api.d.ts` is 12,987 lines and defines 103 named component schemas; `platform.d.ts` is 26,811 lines with 315 named component schemas → `packages/api-types/types/api.d.ts`
- The barrel `index.ts` merges both specs into unified `paths`, `operations`, and `components` interfaces via TypeScript intersection types → `packages/api-types/index.ts`
- Studio imports `api-types` in at least 116 files across `apps/studio/`, primarily in `data/` query hooks and `pages/api/` proxy routes → `apps/studio/data/`
- Studio's `fetchers.ts` creates an `openapi-fetch` client parameterized with the merged `paths` type, giving every `get()`, `post()`, etc. call full path and response type inference → `apps/studio/data/fetchers.ts`
- The v1 API paths are dominated by the `projects` resource group (88 of 107 paths), followed by `organizations` (6) and `branches` (6) → `packages/api-types/types/api.d.ts`
- The platform API's largest resource groups are `projects` (64 paths), `organizations` (56), `storage` (24), `replication` (19), and `integrations` (15) → `packages/api-types/types/platform.d.ts`

## Source of Truth

The upstream OpenAPI specifications live on the Supabase platform API server (not in this repository). The `api-types` package is a **derived artifact** -- its `.d.ts` files are regenerated by running `pnpm codegen` inside `packages/api-types/`, which fetches the live specs via the URLs configured in `redocly.yaml` and runs `openapi-typescript` followed by Prettier formatting.

| Spec | Fetch URL | Output File | Paths | Operations |
|------|-----------|-------------|-------|------------|
| v1 Management API | `http://localhost:8080/api/v1-json` | `types/api.d.ts` | 107 | 163 |
| Platform API | `http://localhost:8080/api/platform-json` | `types/platform.d.ts` | 253 | 345 |

## Resource Map

### v1 Management API (`/v1/...`)

| Resource Group | Path Count | Examples |
|---------------|-----------|----------|
| projects | 88 | `/v1/projects/{ref}`, `/v1/projects/{ref}/functions`, `/v1/projects/{ref}/config/auth` |
| organizations | 6 | `/v1/organizations/{slug}`, `/v1/organizations/{slug}/members` |
| branches | 6 | `/v1/branches/{branch_id_or_ref}`, `/v1/branches/{branch_id_or_ref}/merge` |
| oauth | 4 | `/v1/oauth/authorize`, `/v1/oauth/token` |
| snippets | 2 | `/v1/snippets`, `/v1/snippets/{id}` |
| profile | 1 | `/v1/profile` |

### Platform API (`/platform/...`)

| Resource Group | Path Count | Examples |
|---------------|-----------|----------|
| projects | 64 | `/platform/projects/{ref}/settings`, `/platform/projects/{ref}/billing` |
| organizations | 56 | `/platform/organizations/{slug}/billing`, `/platform/organizations/{slug}/apps` |
| storage | 24 | `/platform/storage/{ref}/buckets`, `/platform/storage/{ref}/objects` |
| replication | 19 | `/platform/replication/{ref}/destinations`, `/platform/replication/{ref}/pipelines` |
| integrations | 15 | `/platform/integrations/github/connections`, `/platform/integrations/vercel` |
| pg-meta | 11 | `/platform/pg-meta/{ref}/tables`, `/platform/pg-meta/{ref}/columns` |
| auth | 11 | `/platform/auth/{ref}/config`, `/platform/auth/{ref}/users` |
| database | 10 | `/platform/database/{ref}/backups`, `/platform/database/{ref}/clone` |
| telemetry | 8 | `/platform/telemetry/event` |
| profile | 8 | `/platform/profile` |

## Worked Examples

### How a Studio query hook uses api-types

The edge functions listing query (`apps/studio/data/edge-functions/edge-functions-query.ts`) demonstrates the standard pattern:

```typescript
// 1. Import the components type from api-types
import { components } from 'api-types'

// 2. Extract a named schema for the response type
export type EdgeFunctionsResponse = components['schemas']['FunctionResponse']

// 3. Use the typed fetcher — path string is checked against api-types paths
const { data, error } = await get('/v1/projects/{ref}/functions', {
  params: { path: { ref: projectRef } },
  signal,
})
```

The `get()` function is created via `openapi-fetch`'s `createClient<paths>()` in `fetchers.ts`. Because `paths` is the merged interface from `api-types`, the path string `/v1/projects/{ref}/functions` is validated at compile time, and the return type of `data` is automatically inferred from the operation's response schema. This eliminates manual type casting and catches path typos or parameter mismatches during the build.

## Related

- [[API-STUDIO]] -- parent artifact; this package provides the type foundation for Studio's API layer
- [[SYS-STUDIO]] -- Studio system overview; api-types feeds type safety into Studio's data layer
- [[CON-STUDIO-SERVICES]] -- contract governing how Studio communicates with backend services via these typed APIs
