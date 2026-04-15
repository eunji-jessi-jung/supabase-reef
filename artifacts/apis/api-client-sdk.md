---
id: "API-CLIENT-SDK"
type: "api"
title: "Supabase JS SDK Public API"
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
known_unknowns:
  - "Complete method signatures for each sub-client"
  - "Full PostgREST query builder API surface"
  - "Edge functions client capabilities"
tags:
  - client-sdk
  - typescript
  - api
aliases: []
relates_to:
  - type: "parent"
    target: "[[SYS-CLIENT-SDK]]"
sources:
  - category: "implementation"
    type: "github"
    ref: "packages/core/supabase-js/src/SupabaseClient.ts"
notes: ""
---

## Overview

The SDK exposes a unified client (`createClient()`) that provides access to all Supabase services through a fluent TypeScript API. Each sub-client handles one service domain. The API is designed for isomorphic usage across Node.js, browsers, and edge runtimes.

## Key Facts

- Main entry: `createClient(url, key, options)` returns a SupabaseClient instance → `packages/core/supabase-js/src/`
- Auth client: `supabase.auth.*` for sign-up, sign-in, sign-out, session management, MFA → `packages/core/auth-js/src/`
- Database client: `supabase.from('table').*` returns PostgREST query builder with full select/insert/update/delete/rpc → `packages/core/postgrest-js/src/`
- Realtime client: `supabase.channel('topic').*` for broadcast, presence, postgres changes → `packages/core/realtime-js/src/`
- Storage client: `supabase.storage.from('bucket').*` for upload, download, list, remove, signed URLs → `packages/core/storage-js/src/`
- Functions client: `supabase.functions.invoke('name')` for edge function invocation → `packages/core/functions-js/src/`
- Advanced TypeScript generics infer database schema types from generated types → `packages/core/supabase-js/src/SupabaseClient.ts`

## Source of Truth

SupabaseClient class in `packages/core/supabase-js/src/SupabaseClient.ts`. Each sub-client in its respective `packages/core/{name}/src/` directory.

## Resource Map

| Sub-client | Access Pattern | Service | Key Operations |
|-----------|---------------|---------|----------------|
| auth | `supabase.auth.*` | Auth | signUp, signInWithPassword, signInWithOtp, signOut, getSession, getUser, onAuthStateChange |
| from (PostgREST) | `supabase.from('table').*` | PostgREST | select, insert, update, upsert, delete, rpc |
| realtime | `supabase.channel('topic').*` | Realtime | subscribe, send (broadcast), track (presence), on (postgres_changes) |
| storage | `supabase.storage.from('bucket').*` | Storage | upload, download, list, remove, createSignedUrl, getPublicUrl |
| functions | `supabase.functions.invoke('name')` | Edge Functions | invoke with body and headers |

## Related

- [[SYS-CLIENT-SDK]] -- parent system
