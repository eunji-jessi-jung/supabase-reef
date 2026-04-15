---
id: "GLOSSARY-SUPABASE-REEF"
type: "glossary"
title: "Supabase Platform Unified Glossary"
domain: "supabase-reef"
status: "draft"
last_verified: 2026-04-15
freshness_note: "snorkel-depth scan"
freshness_triggers:
  - "internal/models/"
  - "lib/realtime/"
  - "src/storage/"
  - "apps/studio/"
  - "packages/core/"
known_unknowns:
  - "Some cross-service term disambiguation may need domain expert confirmation"
tags:
  - glossary
  - cross-service
aliases: []
relates_to:
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
  - category: "context"
    type: "documentation"
    ref: "sources/context/decisions/architecture-overview.md"
    notes: "Architecture overview naming PostgREST, Supavisor, Kong, Deno (Edge Functions) as platform components"
notes: ""
---

## Overview

Unified glossary for the Supabase platform, resolving terms that appear across multiple services. Per-service glossaries are authoritative within their scope; this glossary disambiguates where the same term means different things.

## Terms

| Term | Definition | Used In | See Also |
|------|-----------|---------|----------|
| JWT | JSON Web Token issued by Auth and consumed by all services for authentication and RLS. The shared currency of the Supabase platform. | All services | [[CON-AUTH-STORAGE]], [[CON-AUTH-REALTIME]] |
| RLS | Row Level Security — Postgres feature that filters data access based on JWT claims. Used by Auth (its own tables), Storage (bucket/object access), Realtime (broadcast/presence policies), and PostgREST (user data access). | Auth, Storage, Realtime, PostgREST | |
| service_role | A privileged JWT role that bypasses RLS. Used for admin/server-side operations. Must never be exposed to clients. | All services | [[GLOSSARY-AUTH]] |
| Project | A Supabase project instance. In Realtime it is called a "Tenant". In Studio it is the primary organizational unit. In Auth it maps to a single database/configuration. | All services | |
| Tenant | A Supabase project from Realtime's perspective. Each tenant has its own jwt_secret, database connection, and channel configurations. | Realtime | [[GLOSSARY-REALTIME]] |
| Channel | In Realtime: a named topic for WebSocket subscriptions. In the SDK: `supabase.channel('name')`. Not used in Auth or Storage. | Realtime, Client SDK | [[GLOSSARY-REALTIME]] |
| Bucket | A named container for objects in Storage. Has visibility (public/private) and constraint settings. Not used in other services. | Storage | [[GLOSSARY-STORAGE]] |
| PostgREST | REST API server that auto-generates endpoints from Postgres schema. Not in reef sources; part of platform architecture. The SDK's `supabase.from('table')` calls go to PostgREST. | Client SDK, Studio, Platform | -> `sources/context/decisions/architecture-overview.md` |
| GoTrue | Original name of the Auth server, forked from Netlify's GoTrue. Still appears in env vars (GOTRUE_*) and the SDK's GoTrueClient class name. | Auth, Client SDK | [[GLOSSARY-AUTH]], [[GLOSSARY-CLIENT-SDK]] |
| Edge Functions | Deno-based serverless functions (Deno runtime). Not in reef sources; part of platform architecture. Invoked via the SDK's `supabase.functions.invoke()`. | Client SDK, Platform | -> `sources/context/decisions/architecture-overview.md` |
| PKCE | Proof Key for Code Exchange — OAuth2 extension for secure auth flows. Used in Auth and tracked in auth-js. | Auth, Client SDK | [[GLOSSARY-AUTH]] |
| Supavisor | Cloud-native, multi-tenant Postgres connection pooler. Not in reef sources; part of platform architecture. | Platform | -> `sources/context/decisions/architecture-overview.md` |
| Kong | Cloud-native API gateway built on NGINX, used to route and authenticate API requests to Supabase services. Not in reef sources; part of platform architecture. | Platform | -> `sources/context/decisions/architecture-overview.md` |
| postgres-meta | RESTful API for managing Postgres — fetch tables, add roles, run queries. Studio proxies to it via BFF routes. Not in reef sources; part of platform architecture. | Studio, Platform | -> `sources/context/decisions/architecture-overview.md` |

## Disambiguation

| Term | Auth Meaning | Realtime Meaning | Storage Meaning | Risk |
|------|-------------|-----------------|----------------|------|
| Project/Tenant | Implicit — Auth config is per-project | Explicit — `_realtime.tenants` table with per-tenant jwt_secret | Implicit — tenant_id in requests | Using "tenant" when talking to non-Realtime teams may confuse |
| RLS | Protects auth schema tables | Filters broadcast/presence via policies | Filters bucket/object access | Same mechanism but different policy definitions per service |
| Token | JWT issued to users | JWT validated on WebSocket join | JWT validated on HTTP requests | All are the same JWT but validated independently |

## Related

- [[GLOSSARY-AUTH]] -- Auth-specific terms
- [[GLOSSARY-REALTIME]] -- Realtime-specific terms
- [[GLOSSARY-STORAGE]] -- Storage-specific terms
- [[GLOSSARY-STUDIO]] -- Studio-specific terms
- [[GLOSSARY-CLIENT-SDK]] -- Client SDK-specific terms
