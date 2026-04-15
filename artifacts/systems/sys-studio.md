---
id: "SYS-STUDIO"
type: "system"
title: "Supabase Studio"
domain: "studio"
status: "draft"
last_verified: 2026-04-15
freshness_note: "Deep read of apps/studio, packages/pg-meta, packages/api-types on 2026-04-15"
freshness_triggers:
  - "apps/studio/app/"
  - "apps/studio/data/fetchers.ts"
  - "apps/studio/lib/api/"
  - "apps/studio/lib/constants/index.ts"
  - "apps/studio/next.config.ts"
  - "apps/studio/package.json"
  - "packages/api-types/redocly.yaml"
  - "packages/api-types/types/"
  - "packages/pg-meta/src/index.ts"
  - "pnpm-workspace.yaml"
  - "turbo.jsonc"
known_unknowns:
  - "Exact Vercel deployment topology and CDN edge routing for production Studio"
  - "Full set of environment variables required for self-hosted vs platform mode"
  - "How the Stripe integration handles webhook callbacks for billing"
  - "MCP server integration scope and AI assistant feature boundaries"
tags:
  - studio
  - react
  - nextjs
  - typescript
  - turborepo
  - openapi-fetch
  - bff
aliases:
  - "Dashboard"
relates_to:
  - type: "refines"
    target: "[[API-STUDIO]]"
  - type: "refines"
    target: "[[CON-STUDIO-SERVICES]]"
  - type: "refines"
    target: "[[GLOSSARY-STUDIO]]"
  - type: "refines"
    target: "[[PROC-STUDIO-AUTH]]"
  - type: "refines"
    target: "[[RISK-STUDIO-KNOWN-GAPS]]"
  - type: "refines"
    target: "[[SCH-STUDIO-PG-META]]"
  - type: "integrates_with"
    target: "[[SYS-AUTH]]"
  - type: "feeds"
    target: "[[SYS-CLIENT-SDK]]"
  - type: "integrates_with"
    target: "[[SYS-REALTIME]]"
  - type: "integrates_with"
    target: "[[SYS-STORAGE]]"
sources:
  - category: "implementation"
    type: "github"
    ref: "apps/studio/data/fetchers.ts"
  - category: "implementation"
    type: "github"
    ref: "apps/studio/data/query-client.ts"
  - category: "implementation"
    type: "github"
    ref: "apps/studio/lib/api/apiAuthenticate.ts"
  - category: "implementation"
    type: "github"
    ref: "apps/studio/lib/api/apiHelpers.ts"
  - category: "implementation"
    type: "github"
    ref: "apps/studio/lib/api/apiWrapper.ts"
  - category: "implementation"
    type: "github"
    ref: "apps/studio/lib/constants/index.ts"
  - category: "implementation"
    type: "github"
    ref: "apps/studio/lib/gotrue.ts"
  - category: "implementation"
    type: "github"
    ref: "apps/studio/next.config.ts"
  - category: "implementation"
    type: "github"
    ref: "apps/studio/package.json"
  - category: "implementation"
    type: "github"
    ref: "packages/api-types/package.json"
  - category: "implementation"
    type: "github"
    ref: "packages/api-types/redocly.yaml"
  - category: "implementation"
    type: "github"
    ref: "packages/pg-meta/package.json"
  - category: "implementation"
    type: "github"
    ref: "packages/pg-meta/src/index.ts"
  - category: "implementation"
    type: "github"
    ref: "pnpm-workspace.yaml"
  - category: "implementation"
    type: "github"
    ref: "turbo.jsonc"
notes: ""
---

## Overview

Studio is the Supabase admin dashboard -- a React/Next.js application living in a pnpm + Turborepo monorepo. It acts as a **BFF (Backend-for-Frontend)**: the Next.js `pages/api/` layer proxies and augments requests to the Supabase platform API, pg-meta, auth (GoTrue), storage, and other backend services so the React client never talks to them directly. Three shared packages are central to Studio's operation: `packages/pg-meta` (Postgres introspection SQL library), `packages/api-types` (OpenAPI-generated TypeScript types for both the v1 and platform APIs), and `packages/common` (shared auth and telemetry utilities).

## Key Facts

- pnpm 10 + Turborepo monorepo; workspace defined in `pnpm-workspace.yaml` with `apps/*`, `packages/*`, `blocks/*`, `e2e/*` globs → `pnpm-workspace.yaml`
- Turborepo task graph: `build` depends on `^build`, `typecheck` depends on `^typecheck`; cache outputs are `dist/**` and `.next/**` → `turbo.jsonc`
- Studio is a Next.js app (v16.2.3 via catalog) running on port 8082, with `output: 'standalone'` for containerized deployment → `apps/studio/package.json`, `apps/studio/next.config.ts`
- `IS_PLATFORM` flag (`NEXT_PUBLIC_IS_PLATFORM`) switches behavior between hosted Supabase platform and self-hosted mode, affecting auth, redirects, and API routing → `apps/studio/lib/constants/index.ts`
- BFF data-fetching layer uses `openapi-fetch` with auto-generated types from `packages/api-types`; the `client` in `data/fetchers.ts` adds auth headers and pg-meta connection guards via middleware → `apps/studio/data/fetchers.ts`
- `packages/api-types` generates `api.d.ts` (12,987 lines) and `platform.d.ts` (26,811 lines) from two OpenAPI endpoints via `openapi-typescript` and `redocly.yaml` → `packages/api-types/redocly.yaml`, `packages/api-types/package.json`
- `packages/pg-meta` exports 18 introspection modules (tables, columns, functions, policies, roles, views, indexes, triggers, extensions, schemas, types, publications, foreign tables, materialized views, column privileges, table privileges, config, version) plus Studio-specific SQL for advisor, auth, storage, database, table-editor, sql-editor, role-impersonation, and integrations → `packages/pg-meta/src/index.ts`
- `pg-meta` has zero runtime dependencies aside from Zod; uses `pg` only in devDependencies for testing → `packages/pg-meta/package.json`
- Server-side BFF routes in `pages/api/platform/` proxy to backend services with domains: auth, database, integrations, organizations, pg-meta, profile, projects, props, storage, telemetry → `apps/studio/pages/api/platform/`
- Authentication: platform mode validates JWT via GoTrue (`@supabase/auth-js` through `common/gotrue`); `apiAuthenticate` extracts bearer token and calls `getUserClaims` → `apps/studio/lib/api/apiAuthenticate.ts`, `apps/studio/lib/gotrue.ts`
- React Query (TanStack Query v5) manages client-side cache with 60s default stale time, retry logic that respects 429 rate limits with `Retry-After` header, and pathname-specific retry skipping → `apps/studio/data/query-client.ts`
- AI integration: Studio bundles `@ai-sdk/openai`, `@ai-sdk/amazon-bedrock`, `@modelcontextprotocol/sdk`, and `@supabase/mcp-server-supabase` for in-dashboard AI assistant features → `apps/studio/package.json`
- Static asset CDN: production builds upload assets to `frontend-assets.supabase.com` (or `.green` for staging) via `assetPrefix` in `next.config.ts` → `apps/studio/next.config.ts`
- Workspace packages used by Studio: `@supabase/pg-meta`, `ai-commands`, `common`, `config`, `dev-tools`, `icons`, `shared-data`, `ui`, `ui-patterns`, plus `api-types` and `eslint-config-supabase` in devDependencies → `apps/studio/package.json`
- Studio runs Braintrust evals for the AI assistant (`evals/assistant.eval.ts`) and uses `libpg-query` WASM for SQL parsing → `apps/studio/package.json`

## Dependencies

| System / Package | Integration Type | Purpose | Auth Method |
|---|---|---|---|
| [[SYS-AUTH]] (GoTrue) | BFF proxy via `pages/api/platform/auth/` | User authentication, JWT validation, session management | Bearer JWT token |
| [[SYS-STORAGE]] | BFF proxy via `pages/api/platform/storage/` | Bucket and object management UI | Bearer JWT token |
| [[SYS-REALTIME]] | Client SDK (`@supabase/realtime-js`) | Real-time subscriptions for dashboard updates | Bearer JWT token |
| Supabase Platform API | `openapi-fetch` client typed by `api-types` | Project CRUD, org management, billing, analytics | Bearer JWT token via middleware |
| pg-meta backend | BFF proxy via `pages/api/platform/pg-meta/` | Database introspection and SQL execution | `x-connection-encrypted` header + Bearer JWT |
| Stripe | `@stripe/stripe-js` + `@stripe/react-stripe-js` | Billing UI, plan management | Stripe public key |
| Vercel | Deployment platform, asset CDN, feature flags | Hosting, static asset delivery | Vercel env vars |
| PostHog | Analytics tracking via `common` package | User telemetry | PostHog API key |
| Sentry | `@sentry/nextjs` | Error monitoring and reporting | Sentry DSN |

## Does NOT Own

- **Auth service (GoTrue)** -- Studio configures auth providers and displays users, but authentication logic and user storage live in the GoTrue service. Studio only validates JWTs.
- **Storage service** -- Studio provides bucket/object management UI but does not store files. All operations proxy through the Storage API.
- **Realtime service** -- Studio may display realtime config but does not implement the WebSocket broadcast or presence engine.
- **PostgREST / Data API** -- Studio's API explorer and SQL editor generate queries, but PostgREST serves the actual data API to end-user applications.
- **Postgres database operations** -- `pg-meta` reads system catalog metadata only. Actual DDL/DML is executed by the database itself; Studio submits SQL through the pg-meta proxy.
- **Client SDKs** -- Studio is not the `supabase-js` client SDK. It consumes `@supabase/auth-js` and `@supabase/realtime-js` internally but does not publish them.

## Domain Behavior Highlights

- **BFF pattern**: Every backend call from the React client goes through Next.js API routes in `pages/api/`. The `apiWrapper` function provides a global error catch and optional JWT authentication via `apiAuthenticate`. The client never directly reaches GoTrue, pg-meta, or the platform API. → `apps/studio/lib/api/apiWrapper.ts`, `apps/studio/data/fetchers.ts`
- **pg-meta introspection**: The `@supabase/pg-meta` package is a pure TypeScript SQL-generation library (zero runtime deps beyond Zod). It builds parameterized queries against Postgres system catalogs (`pg_catalog`, `information_schema`) and exports Studio-specific SQL helpers for features like the table editor, SQL editor, and role impersonation. → `packages/pg-meta/src/index.ts`
- **api-types contract**: `packages/api-types` auto-generates TypeScript type definitions from two live OpenAPI endpoints (v1 API and platform API) using `openapi-typescript`. The `openapi-fetch` client in `data/fetchers.ts` consumes these types, giving compile-time safety for every API call Studio makes. → `packages/api-types/redocly.yaml`, `apps/studio/data/fetchers.ts`
- **Platform vs self-hosted mode**: The `IS_PLATFORM` constant gates significant behavior differences -- platform mode uses GoTrue auth and external API URLs, while self-hosted mode assumes a local database (`project/default`), injects `SUPABASE_SERVICE_KEY` as apiKey, and keeps React Query online regardless of network status. → `apps/studio/lib/constants/index.ts`, `apps/studio/lib/api/apiHelpers.ts`, `apps/studio/data/query-client.ts`
- **pg-meta connection guard**: Before any `/platform/pg-meta/` request reaches the backend, the `openapi-fetch` middleware validates that `x-connection-encrypted` is present (platform mode) and sets `x-pg-application-name` to a default if missing, failing fast with a 400 if the connection string is absent. → `apps/studio/data/fetchers.ts`

## Runtime Components

| Component | Tech Stack | Purpose | Entry Point |
|---|---|---|---|
| Studio Web App | Next.js 16, React 18, Tailwind CSS | Admin dashboard SPA with SSR/SSG | `apps/studio/` (`next dev -p 8082`) |
| BFF API Layer | Next.js Pages Router API routes | Proxy + auth gateway to platform services | `apps/studio/pages/api/` |
| Data Fetching Layer | openapi-fetch + TanStack Query v5 | Type-safe API client with caching and retry | `apps/studio/data/fetchers.ts` |
| pg-meta Library | TypeScript, Zod | SQL generation for Postgres introspection | `packages/pg-meta/src/index.ts` |
| api-types Package | openapi-typescript, Redocly | Generated TypeScript types from OpenAPI specs | `packages/api-types/types/` |
| AI Assistant | Vercel AI SDK, OpenAI, Bedrock, MCP | In-dashboard AI features and evals | `apps/studio/pages/api/ai/` |
| E2E Test Suite | Playwright | End-to-end testing | `e2e/studio/` |

## Responsibilities

- **Owns**: Admin dashboard UI, BFF API proxy layer, Postgres introspection SQL generation (pg-meta), platform API type contracts (api-types), project management UI, SQL editor, database explorer, auth provider configuration UI, storage bucket management UI, AI assistant integration
- **Does not own**: Backend API implementation (delegates to Auth, Storage, Realtime, PostgREST), user-facing client SDK, actual Postgres database operations (pg-meta generates SQL metadata queries only), billing backend (Stripe webhooks and subscription logic)

## Core Concepts

- **Project**: A Supabase project instance, the primary organizational unit in Studio. Platform mode manages multiple projects; self-hosted redirects to `project/default`.
- **pg-meta**: Library that generates SQL queries against Postgres system catalogs to provide metadata about tables, columns, functions, policies, etc. Zero runtime deps beyond Zod.
- **api-types**: TypeScript type definitions auto-generated from the Supabase platform API OpenAPI specs (v1 and platform endpoints) via `openapi-typescript`.
- **BFF (Backend-for-Frontend)**: Studio's Next.js API routes act as an intermediary, adding auth headers, connection guards, and error formatting before forwarding requests to backend services.
- **Data hooks**: React Query hooks in `apps/studio/data/` organized by domain (30+ subdirectories covering auth, database, storage, analytics, etc.), using a queryOptions pattern for data fetching.
- **IS_PLATFORM**: Boolean flag that switches between Supabase-hosted platform mode (external APIs, GoTrue auth, multi-project) and self-hosted mode (local DB, service key auth, single project).

## Related

- [[SCH-STUDIO-PG-META]] -- pg-meta introspection model and entity catalog
- [[API-STUDIO]] -- Studio API surface and BFF route inventory
- [[PROC-STUDIO-AUTH]] -- authentication and session management flow
- [[GLOSSARY-STUDIO]] -- domain terminology
- [[RISK-STUDIO-KNOWN-GAPS]] -- known risks and unknowns
- [[CON-STUDIO-SERVICES]] -- integration contract between Studio and backend services
- [[SYS-AUTH]] -- authenticates dashboard users via GoTrue JWT validation
- [[SYS-STORAGE]] -- manages storage buckets/objects via Studio UI proxy
- [[SYS-REALTIME]] -- manages realtime subscriptions via Studio
- [[SYS-CLIENT-SDK]] -- Studio feeds project config consumed by client SDKs
