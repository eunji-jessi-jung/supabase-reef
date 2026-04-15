---
id: "PROC-STUDIO-MULTI-APP-COMPARISON"
type: "process"
title: "Studio Multi-App Comparison: Studio vs Api-Types vs Pg-Meta"
domain: "studio"
status: draft
last_verified: 2026-04-15
freshness_note: "scuba-depth multi-app comparison"
freshness_triggers:
  - "apps/studio/package.json"
  - "apps/studio/data/fetchers.ts"
  - "apps/studio/data/api.d.ts"
  - "apps/studio/vitest.config.ts"
  - "apps/studio/tsconfig.json"
  - "packages/api-types/package.json"
  - "packages/api-types/index.ts"
  - "packages/api-types/redocly.yaml"
  - "packages/pg-meta/package.json"
  - "packages/pg-meta/src/index.ts"
  - "packages/pg-meta/src/query/"
  - "turbo.jsonc"
known_unknowns:
  - "Whether api-types codegen is run in CI automatically or only manually by developers"
  - "How pg-meta Zod schemas are kept in sync with the actual Postgres system catalog across PG versions"
  - "Whether any packages outside Studio also consume api-types (e.g., docs, www, or CLI)"
  - "Full list of Studio components that import pg-meta Zod types directly for local form validation"
tags:
  - comparison
  - multi-app
aliases: []
relates_to:
  - type: "parent"
    target: "[[SYS-STUDIO]]"
  - type: "refines"
    target: "[[SCH-STUDIO-PG-META]]"
  - type: "refines"
    target: "[[API-STUDIO]]"
sources:
  - category: implementation
    type: github
    ref: "apps/studio/package.json"
    notes: "Studio dependency declarations — workspace refs to pg-meta and api-types"
  - category: implementation
    type: github
    ref: "apps/studio/data/fetchers.ts"
    notes: "openapi-fetch client consuming api-types paths; pg-meta constant import for connection guard"
  - category: implementation
    type: github
    ref: "apps/studio/data/api.d.ts"
    notes: "Re-export shim that forwards all types from api-types package"
  - category: implementation
    type: github
    ref: "apps/studio/vitest.config.ts"
    notes: "Studio test config — jsdom environment, React plugin, coverage on lib/"
  - category: implementation
    type: github
    ref: "apps/studio/tsconfig.json"
    notes: "Extends tsconfig/nextjs.json with path aliases for @/ and @ui/"
  - category: implementation
    type: github
    ref: "packages/api-types/package.json"
    notes: "Codegen-only package — openapi-typescript + prettier, no runtime deps"
  - category: implementation
    type: github
    ref: "packages/api-types/index.ts"
    notes: "Merges api and platform types into unified paths/operations/components interfaces"
  - category: implementation
    type: github
    ref: "packages/api-types/redocly.yaml"
    notes: "Two OpenAPI source endpoints: v1 API and platform API on localhost:8080"
  - category: implementation
    type: github
    ref: "packages/pg-meta/package.json"
    notes: "Zod-only runtime dep; pg only in devDeps for integration tests with Docker"
  - category: implementation
    type: github
    ref: "packages/pg-meta/src/index.ts"
    notes: "18 introspection modules + 8 studio-specific SQL re-exports + pg-format utilities"
  - category: implementation
    type: github
    ref: "packages/pg-meta/src/query/index.ts"
    notes: "Query builder exports: Query, QueryAction, QueryFilter, QueryModifier, table-row-query"
  - category: implementation
    type: github
    ref: "packages/pg-meta/src/pg-format/index.ts"
    notes: "SafeSqlFragment branded type and ident/literal/keyword/safeSql escaping functions"
  - category: implementation
    type: github
    ref: "packages/pg-meta/tsconfig.json"
    notes: "Extends tsconfig/base.json — no framework-specific config"
  - category: implementation
    type: github
    ref: "turbo.jsonc"
    notes: "Build task depends on ^build; typecheck depends on ^typecheck — enforces package build order"
notes: ""
---

## Overview

The Studio service comprises three sub-apps within the Supabase pnpm + Turborepo monorepo: `apps/studio` (the Next.js dashboard application), `packages/api-types` (auto-generated TypeScript type definitions from OpenAPI specs), and `packages/pg-meta` (a pure TypeScript SQL-generation library for Postgres introspection). These three packages have a clear dependency hierarchy: Studio is the consumer that depends on both api-types and pg-meta, while api-types and pg-meta are independent of each other and serve fundamentally different purposes -- one provides compile-time type safety for HTTP API calls, the other provides runtime SQL generation for database introspection.

## Key Facts

- **Dependency direction is strictly one-way**: Studio declares `"@supabase/pg-meta": "workspace:*"` in dependencies and `"api-types": "workspace:*"` in devDependencies; neither pg-meta nor api-types depends on the other or on Studio → `apps/studio/package.json`, `packages/api-types/package.json`, `packages/pg-meta/package.json`
- **api-types is a build-time-only package** with zero runtime dependencies; it has only three devDependencies (`openapi-typescript`, `prettier`, `typescript`) and its sole script is `codegen` which generates `.d.ts` files from two OpenAPI endpoints → `packages/api-types/package.json`
- **pg-meta has exactly one runtime dependency (Zod)** and uses `pg` only in devDependencies for integration testing against a Docker-managed Postgres instance via `docker compose` → `packages/pg-meta/package.json`
- **Studio is a heavyweight Next.js application** with 100+ runtime dependencies including React, TanStack Query, Monaco editor, AI SDK (OpenAI + Bedrock + MCP), Stripe, Sentry, recharts, framer-motion, and numerous workspace packages (ui, ui-patterns, icons, common, config, shared-data, ai-commands, dev-tools) → `apps/studio/package.json`
- **api-types merges two OpenAPI specs into unified interfaces**: `index.ts` imports from `types/api.d.ts` (12,987 lines, v1 API) and `types/platform.d.ts` (26,811 lines, platform API), then re-exports merged `paths`, `operations`, and `components` interfaces that combine both specs → `packages/api-types/index.ts`, `packages/api-types/types/`
- **Studio consumes api-types through an `openapi-fetch` client** created in `data/fetchers.ts` with `createClient<paths>()`, plus a local re-export shim at `data/api.d.ts` that does `export type * from 'api-types'` — this gives all 91+ data-fetching files type-safe HTTP method wrappers → `apps/studio/data/fetchers.ts`, `apps/studio/data/api.d.ts`
- **Studio consumes pg-meta in two distinct ways**: (1) importing SQL-generation functions like `getUsersCountSQL`, `getEntityTypesSQL`, `getCreateFDWSql` from the studio-specific SQL modules for building queries sent to the pg-meta backend, and (2) importing the `Query` class and `wrapWithTransaction` utility from the query builder for constructing parameterized table-row queries — totaling 179 import occurrences across 161 files → `apps/studio/data/` (various files), `packages/pg-meta/src/index.ts`
- **pg-meta exports a SafeSqlFragment branded type system** (`string & { readonly __safeSqlFragmentBrand: never }`) with `ident()`, `literal()`, `keyword()`, and `safeSql` tagged-template functions to prevent SQL injection at the type level — Studio imports these for composing safe SQL in its data layer → `packages/pg-meta/src/pg-format/index.ts`, `apps/studio/components/interfaces/SQLEditor/SQLEditor.tsx`
- **pg-meta contains 5,633 lines of Studio-specific SQL** in `src/sql/studio/` across 8 subdirectories (advisor, auth, storage, database, table-editor, sql-editor, role-impersonation, integrations), which are re-exported from `src/index.ts` and constitute the primary SQL surface that Studio's data hooks consume → `packages/pg-meta/src/sql/studio/`, `packages/pg-meta/src/index.ts`
- **Testing stacks differ significantly**: Studio uses Vitest with jsdom environment, React Testing Library, MSW for mocking, and coverage scoped to `lib/**/*.ts`; pg-meta uses Vitest with real Postgres via Docker Compose (`test/db/`) for integration tests that execute generated SQL against an actual database; api-types has no tests at all (codegen output only) → `apps/studio/vitest.config.ts`, `packages/pg-meta/package.json`
- **TypeScript configs diverge by purpose**: Studio extends `tsconfig/nextjs.json` with `@/` and `@ui/` path aliases and Next.js plugin; pg-meta extends `tsconfig/base.json` with no framework config; api-types has no tsconfig — its output is raw `.d.ts` files consumed by other packages → `apps/studio/tsconfig.json`, `packages/pg-meta/tsconfig.json`
- **Build orchestration via Turborepo**: the `build` task depends on `^build` (topological), meaning api-types and pg-meta build before Studio; api-types has no build script (types are pre-generated by `codegen`), pg-meta has no build script (consumed directly via `main: ./src/index.ts`), so Studio's `next build` is the only real build step → `turbo.jsonc`, `packages/api-types/package.json`, `packages/pg-meta/package.json`
- **pg-meta provides 18 generic introspection modules** (tables, columns, functions, policies, roles, views, indexes, triggers, extensions, schemas, types, publications, foreign-tables, materialized-views, column-privileges, table-privileges, config, version) plus a query builder (`Query`, `QueryAction`, `QueryFilter`, `QueryModifier`, `table-row-query`) — the generic modules return `{ sql, zod }` tuples for the caller to execute and validate → `packages/pg-meta/src/index.ts`, `packages/pg-meta/src/pg-meta-tables.ts`
- **Shared constant between pg-meta and Studio**: `DEFAULT_PLATFORM_APPLICATION_NAME` ("supabase/dashboard") is defined in `packages/pg-meta/src/constants.ts` and imported by Studio's `data/fetchers.ts` to set the `x-pg-application-name` header on pg-meta requests — this is the only non-type, non-SQL export from pg-meta that Studio uses for runtime HTTP behavior → `packages/pg-meta/src/constants.ts`, `apps/studio/data/fetchers.ts`
- **api-types codegen requires a running local API server** on `localhost:8080` serving `/api/v1-json` and `/api/platform-json` endpoints, as configured in `redocly.yaml` — this means type generation is an offline developer task, not part of the standard build pipeline → `packages/api-types/redocly.yaml`

## Comparison Matrix

| Dimension | apps/studio | packages/api-types | packages/pg-meta |
|---|---|---|---|
| **Purpose** | Full admin dashboard (UI + BFF proxy) | TypeScript type definitions for HTTP APIs | SQL generation for Postgres introspection |
| **Runtime deps** | 100+ (React, Next.js, TanStack Query, Monaco, AI SDK, Stripe, etc.) | 0 | 1 (Zod) |
| **Dev deps** | 40+ (testing, linting, codegen) | 3 (openapi-typescript, prettier, typescript) | 7 (vitest, pg, typescript, etc.) |
| **Language** | TypeScript + TSX (React) | TypeScript (generated .d.ts) | TypeScript |
| **Framework** | Next.js 16 (Pages Router) | None | None |
| **Build tool** | Next.js (`next build`) | None (codegen via `openapi-typescript`) | None (consumed via `main: ./src/index.ts`) |
| **Test framework** | Vitest + jsdom + React Testing Library + MSW | None | Vitest + real Postgres (Docker) |
| **Test environment** | Simulated browser (jsdom) | N/A | Real database (Docker Compose) |
| **Output** | Standalone Next.js app (.next/) | `.d.ts` type definition files | TypeScript source (no compiled output) |
| **Lines of code (approx)** | Large (full app) | ~40K (generated) | ~10K (hand-written) |
| **Package name** | `studio` (private) | `api-types` | `@supabase/pg-meta` |
| **Published to npm** | No (private) | No | No (workspace-only) |
| **Zod usage** | Indirect (via pg-meta) + direct for forms | None | Core — every introspection module exports Zod schemas |
| **Consumed by** | End users (browser) | Studio (compile-time) | Studio (runtime SQL generation) |

## Overlap and Boundaries

- **Type overlap**: api-types defines HTTP response shapes (e.g., project, organization, billing types), while pg-meta defines Postgres catalog shapes (e.g., `PGTable`, `PGColumn`, `PGFunction` via Zod). Studio consumes both, but they model entirely different domains with no shared type definitions.
- **SQL execution boundary**: pg-meta generates SQL strings but never executes them. Studio's BFF layer sends pg-meta-generated SQL to the pg-meta backend service via HTTP. The pg-meta package and the pg-meta backend service are separate — the package is a library, the service is a server.
- **Validation boundary**: pg-meta owns Zod schemas for database entity validation; api-types owns TypeScript interfaces for API response validation. Studio uses both — pg-meta Zod schemas validate database query results, api-types interfaces provide compile-time guarantees on API fetcher calls.
- **No cross-dependency between packages**: api-types and pg-meta are completely independent. They share no code, no types, and no build dependencies. Their only connection is that Studio consumes both.

## Related

- [[SYS-STUDIO]] -- parent system artifact covering the full Studio architecture
- [[SCH-STUDIO-PG-META]] -- detailed schema of pg-meta's introspection model
- [[API-STUDIO]] -- Studio's BFF API route inventory (consumes both api-types and pg-meta)
