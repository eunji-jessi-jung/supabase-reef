---
id: "API-STUDIO"
type: "api"
title: "Studio API Surface"
domain: "studio"
status: "draft"
last_verified: 2026-04-15
freshness_note: "snorkel-depth scan"
freshness_triggers:
  - "apps/studio/app/api/"
  - "apps/studio/data/"
  - "packages/api-types/types/"
known_unknowns:
  - "Complete list of Next.js API routes in apps/studio/app/api/"
  - "How Studio proxies to backend services (Auth, Storage, Realtime, PostgREST)"
  - "Full platform API type coverage in api-types"
tags:
  - studio
  - rest-api
  - nextjs
aliases: []
relates_to:
  - type: "parent"
    target: "[[SYS-STUDIO]]"
  - type: "depends_on"
    target: "[[SCH-STUDIO-PG-META]]"
sources:
  - category: "implementation"
    type: "github"
    ref: "apps/studio/app/api/route.ts"
  - category: "implementation"
    type: "github"
    ref: "packages/api-types/types/api.d.ts"
notes: ""
---

## Overview

Studio's API surface has two layers: (1) Next.js API routes in `apps/studio/app/api/` that serve as BFF (backend-for-frontend) endpoints proxying to Supabase platform services, and (2) the client-side data layer using React Query hooks in `apps/studio/data/` that call both the BFF routes and direct service endpoints. The `packages/api-types` package provides TypeScript interfaces for `api.d.ts` (service-level APIs) and `platform.d.ts` (management platform APIs).

## Key Facts

- Next.js API routes in `apps/studio/app/api/` serve as BFF layer → `apps/studio/app/api/`
- Platform API types: `api.d.ts` and `platform.d.ts` define typed contracts → `packages/api-types/types/`
- Data layer organized by domain with 30+ query/mutation modules → `apps/studio/data/`
- Domain-specific data modules include: auth, database, storage, analytics, branches, api-keys, permissions, organizations → `apps/studio/data/`
- AI integration endpoint at `apps/studio/pages/api/ai/docs.ts` → `apps/studio/pages/api/ai/`

## Source of Truth

API types in `packages/api-types/types/`. Data hooks in `apps/studio/data/`. BFF routes in `apps/studio/app/api/`.

## Resource Map

### Data Layer Domains

| Domain | Module Path | Description |
|--------|-----------|-------------|
| Auth | `data/auth/` | Auth provider configuration, user management |
| Database | `data/database/` | Table, column, function, trigger, index management |
| Database Policies | `data/database-policies/` | RLS policy CRUD |
| Database Extensions | `data/database-extensions/` | Extension management |
| Storage | `data/storage/` | Bucket and object management |
| Analytics | `data/analytics/` | Infrastructure monitoring queries |
| API Keys | `data/api-keys/` | Project API key management |
| Permissions | `data/permissions/` | Permission queries |
| Organizations | `data/organizations/` | Org-level management |
| Branches | `data/branches/` | Database branching |
| Content | `data/content/` | SQL snippets and saved content |
| Config | `data/config/` | Project configuration |

## Related

- [[SYS-STUDIO]] -- parent system
- [[SCH-STUDIO-PG-META]] -- pg-meta introspection model
