---
id: "SYS-CLIENT-SDK"
type: "system"
title: "Supabase JS SDK"
domain: "client-sdk"
status: "draft"
last_verified: 2026-04-15
freshness_note: "snorkel-depth scan"
freshness_triggers:
  - "packages/core/supabase-js/src/SupabaseClient.ts"
  - "packages/core/auth-js/src/"
  - "packages/core/postgrest-js/src/"
  - "packages/core/realtime-js/src/"
  - "packages/core/storage-js/src/"
  - "packages/core/functions-js/src/"
  - "package.json"
known_unknowns:
  - "Exact version compatibility matrix between SDK and server-side services"
  - "How the fixed versioning mode interacts with independent sub-package changes"
  - "Full TypeScript generic type system for database schema inference"
  - "Cross-platform behavior differences (Node, Deno, Bun, browser)"
tags:
  - client-sdk
  - typescript
  - npm
  - monorepo
aliases:
  - "supabase-js"
  - "@supabase/supabase-js"
relates_to:
  - type: "depends_on"
    target: "[[SYS-AUTH]]"
  - type: "depends_on"
    target: "[[SYS-REALTIME]]"
  - type: "depends_on"
    target: "[[SYS-STORAGE]]"
  - type: "refines"
    target: "[[API-CLIENT-SDK]]"
  - type: "refines"
    target: "[[PROC-CLIENT-SDK-AUTH]]"
  - type: "refines"
    target: "[[GLOSSARY-CLIENT-SDK]]"
  - type: "refines"
    target: "[[RISK-CLIENT-SDK-KNOWN-GAPS]]"
sources:
  - category: "implementation"
    type: "github"
    ref: "packages/core/supabase-js/src/SupabaseClient.ts"
  - category: "implementation"
    type: "github"
    ref: "package.json"
notes: ""
---

## Overview

The Supabase JS SDK is a TypeScript monorepo (Nx-managed) providing the unified client interface to all Supabase services. The main `@supabase/supabase-js` package composes five sub-client libraries: auth-js (authentication), postgrest-js (database via PostgREST), realtime-js (WebSocket messaging), storage-js (file storage), and functions-js (edge functions). All packages share a fixed version number and release together.

## Key Facts

- Nx monorepo with 6 packages under `packages/core/` → `packages/core/`
- SupabaseClient composes auth, postgrest, realtime, storage, and functions sub-clients → `packages/core/supabase-js/src/SupabaseClient.ts`
- Fixed versioning: all packages share identical version numbers (e.g., 2.80.0) → `nx.json`
- Hybrid release model: automated canary releases on every master commit, manual stable promotions → `.github/workflows/`
- Cross-platform support: Node.js, Deno, Bun, browser, React Native (Expo), Next.js SSR → `packages/core/supabase-js/`
- Conventional commits enforced via commitlint → `commitlint.config.js`
- Auth-js requires Docker for integration tests (GoTrue + Postgres) → from CLAUDE.md
- Internal dependencies use `*` protocol, resolved to actual version at release time → `packages/core/supabase-js/package.json`
- TypeScript project references for incremental builds → `tsconfig.base.json`

## Responsibilities

- **Owns**: Client-side API for all Supabase services, session management in the browser/runtime, token injection across sub-clients, TypeScript type safety for database operations
- **Does not own**: Server-side service implementations, JWT issuance, RLS policy evaluation, actual data storage

## Core Concepts

- **SupabaseClient**: The main entry point. Instantiated with project URL and API key. Exposes `.auth`, `.from()` (database), `.realtime`, `.storage`, `.functions` sub-clients.
- **Sub-client**: Each service has its own client library that can also be used independently (@supabase/auth-js, etc.)
- **fetchWithAuth**: Cross-cutting auth integration that injects the current session token into all sub-client requests.
- **Fixed versioning**: All 6 packages always share the same version number, simplifying dependency management.

## Related

- [[API-CLIENT-SDK]] -- public API surface
- [[PROC-CLIENT-SDK-AUTH]] -- auth token management
- [[GLOSSARY-CLIENT-SDK]] -- domain terminology
- [[RISK-CLIENT-SDK-KNOWN-GAPS]] -- known risks
- [[SYS-AUTH]] -- auth-js wraps the Auth API
- [[SYS-REALTIME]] -- realtime-js wraps the Realtime WebSocket
- [[SYS-STORAGE]] -- storage-js wraps the Storage API
