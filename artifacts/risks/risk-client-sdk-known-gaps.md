---
id: "RISK-CLIENT-SDK-KNOWN-GAPS"
type: "risk"
title: "Client SDK Known Risks, Gaps, and Backlog Candidates"
domain: "supabase-reef"
status: draft
last_verified: 2026-04-15
freshness_note: "Deep scan of TODO/FIXME/HACK signals and @deprecated markers across all 6 core packages in the supabase-js monorepo"
freshness_triggers:
  - "packages/core/auth-js/src/"
  - "packages/core/functions-js/src/"
  - "packages/core/postgrest-js/src/"
  - "packages/core/realtime-js/src/"
  - "packages/core/storage-js/src/"
  - "packages/core/supabase-js/src/"
known_unknowns:
  - "Whether v3 breaking changes (noted in postgrest-js TODOs) are planned for a specific timeline"
  - "Impact of the single GoTrueClient TODO in production — line 1472 note says 'flatten type' without context"
tags:
  - risk-catalog
  - client-sdk
aliases: []
relates_to:
  - type: "parent"
    target: "[[SYS-CLIENT-SDK]]"
  - type: "refines"
    target: "[[PROC-CLIENT-SDK-ERROR-HANDLING]]"
sources:
  - category: implementation
    type: github
    ref: "packages/core/auth-js/src/GoTrueClient.ts"
    notes: "1 production TODO, plus deprecated async onAuthStateChange"
  - category: implementation
    type: github
    ref: "packages/core/auth-js/src/lib/types.ts"
    notes: "2 deprecated type properties"
  - category: implementation
    type: github
    ref: "packages/core/functions-js/src/edge-runtime.d.ts"
    notes: "TODO for missing type definitions"
  - category: implementation
    type: github
    ref: "packages/core/postgrest-js/src/PostgrestBuilder.ts"
    notes: "Deprecated returns() method and TODO for shouldThrowOnError"
  - category: implementation
    type: github
    ref: "packages/core/postgrest-js/src/PostgrestQueryBuilder.ts"
    notes: "2 v3 TODOs for defaultToNull consistency"
  - category: implementation
    type: github
    ref: "packages/core/postgrest-js/src/select-query-parser/result.ts"
    notes: "3 TODOs in type-level select query parser"
  - category: implementation
    type: github
    ref: "packages/core/realtime-js/src/lib/transformers.ts"
    notes: "TODO for better Postgres array data separation"
  - category: implementation
    type: github
    ref: "packages/core/storage-js/src/lib/types.ts"
    notes: "TODO for missing metadata props in API swagger"
notes: ""
---

## Description

Catalogs risk signals found across all 6 packages in the Client SDK monorepo. Production source code has 15 TODO/FIXME markers, plus 8 `@deprecated` markers indicating API surface evolution. The PostgREST test suite has an additional 41 TODOs for test coverage expansion.

## Key Facts

- SupabaseClient has 1 TODO noting the `db_schema` generic type parameter cannot be set from `ClientOptions` yet, requiring callers to use the type parameter approach instead → `packages/core/supabase-js/src/SupabaseClient.ts`
- GoTrueClient has 1 production TODO (`flatten type` at line 1472) and a `@deprecated` async `onAuthStateChange` overload warning about deadlock potential with async callbacks → `packages/core/auth-js/src/GoTrueClient.ts`
- Auth-js types has 2 `@deprecated` properties in its type definitions → `packages/core/auth-js/src/lib/types.ts`
- GoTrueClient JWKS endpoint has a `@deprecated` option note: "Please use options.jwks instead" of the previous JWT verification approach → `packages/core/auth-js/src/GoTrueClient.ts`
- PostgREST has 2 `TODO(v3)` markers for making `defaultToNull` behavior consistent between single and bulk insert/upsert operations — this is deferred to a major version → `packages/core/postgrest-js/src/PostgrestQueryBuilder.ts`
- PostgREST select query parser has 3 TODOs in the type-level result computation, including an unexplained necessity for comparing `type` property and a PostgREST v12/v13 inconsistency workaround → `packages/core/postgrest-js/src/select-query-parser/result.ts`
- PostgREST `returns()` method and `foreignTable` option in `order()` are deprecated in favor of `overrideTypes()` and `referencedTable` respectively → `packages/core/postgrest-js/src/PostgrestTransformBuilder.ts` and `packages/core/postgrest-js/src/PostgrestBuilder.ts`
- PostgREST test suite has 41 TODO markers, concentrated in relationship join testing, resource embedding, and advanced RPC scenarios — these are test planning markers not production issues → `packages/core/postgrest-js/test/`
- Realtime transformers has a TODO for finding a better solution to separate Postgres array data from other data types in change payloads → `packages/core/realtime-js/src/lib/transformers.ts`
- Storage types has a TODO noting missing metadata properties in the API swagger documentation, meaning the TypeScript types may not match the actual API response → `packages/core/storage-js/src/lib/types.ts`
- Functions edge-runtime type definitions have a TODO for future type definitions that are not yet provided → `packages/core/functions-js/src/edge-runtime.d.ts`
- PostgREST `PostgrestClient` has a TODO to add back `shouldThrowOnError` once typings are figured out, suggesting the error mode configuration is incomplete at the client level → `packages/core/postgrest-js/src/PostgrestClient.ts`

## Findings

| # | Location | Signal | Theme | Severity |
|---|----------|--------|-------|----------|
| 1 | `supabase-js/src/SupabaseClient.ts` | TODO: Allow db_schema from ClientOptions | API ergonomics | low |
| 2 | `auth-js/src/GoTrueClient.ts` | TODO: flatten type; deprecated async onAuthStateChange | Session management | medium |
| 3 | `postgrest-js/src/PostgrestQueryBuilder.ts` | 2 TODO(v3): defaultToNull consistency | API consistency | medium |
| 4 | `postgrest-js/src/select-query-parser/result.ts` | 3 TODOs in type-level parser | Type safety | medium |
| 5 | `postgrest-js/src/PostgrestClient.ts` | TODO: Add back shouldThrowOnError | Error mode config | low |
| 6 | `postgrest-js/src/PostgrestTransformBuilder.ts` | 2 @deprecated methods (returns, foreignTable) | API evolution | low |
| 7 | `realtime-js/src/lib/transformers.ts` | TODO: better Postgres array separation | Data handling | medium |
| 8 | `storage-js/src/lib/types.ts` | TODO: missing metadata props in swagger | Type accuracy | low |
| 9 | `functions-js/src/edge-runtime.d.ts` | TODO: missing type defs | Type completeness | low |
| 10 | `postgrest-js/test/` | 41 TODOs for test coverage | Test planning | low |

## Themes

### PostgREST v3 Migration Debt (4 signals, medium)

Multiple TODOs are explicitly tagged `TODO(v3)`, indicating known inconsistencies that require breaking changes. The `defaultToNull` behavior differs between single and bulk insert/upsert, and the select query parser has workarounds for PostgREST v12/v13 differences.

**Impact if unaddressed:** Developers encounter inconsistent behavior between single and bulk operations. Type inference may produce incorrect results for certain query patterns.

### Deprecated API Surface (8 markers, low-medium)

Eight `@deprecated` markers span auth-js types, PostgREST transform methods, and GoTrueClient overloads. These represent API evolution where new patterns exist but old ones must be maintained for backward compatibility.

**Impact if unaddressed:** Deprecated methods remain in use, increasing maintenance burden. The async `onAuthStateChange` overload specifically warns about deadlock potential.

### Realtime Data Transformation (1 signal, medium)

The TODO in `transformers.ts` for better Postgres array separation suggests the current approach may not correctly handle all Postgres array types in change payloads.

**Impact if unaddressed:** Edge cases in Postgres array data within Realtime change events may be incorrectly parsed or serialized.

### Test Coverage Gaps (41 signals in tests, low)

PostgREST test files have 41 TODOs concentrated in relationship joins, resource embedding, and RPC testing. These are planning markers for future test expansion.

**Impact if unaddressed:** Edge cases in query building, especially around resource embedding and relationship joins, lack automated test coverage.

## Related

- [[SYS-CLIENT-SDK]] — parent system
- [[PROC-CLIENT-SDK-ERROR-HANDLING]] — error handling gaps are a subset of these risks
