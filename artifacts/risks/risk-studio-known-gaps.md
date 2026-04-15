---
id: "RISK-STUDIO-KNOWN-GAPS"
type: "risk"
title: "Studio Known Risks, Gaps, and Backlog Candidates"
domain: "supabase-reef"
status: draft
last_verified: 2026-04-15
freshness_note: "Deep scan of TODO/FIXME/HACK signals across apps/studio/ — 88 signals in components, plus data and lib directories"
freshness_triggers:
  - "apps/studio/components/interfaces/Auth/AuthProvidersFormValidation.tsx"
  - "apps/studio/components/to-be-cleaned/"
  - "apps/studio/data/analytics/"
  - "apps/studio/data/fetchers.ts"
  - "apps/studio/data/permissions/permissions-query.ts"
  - "apps/studio/hooks/ui/useFlag.ts"
  - "apps/studio/lib/sql-parameters.ts"
  - "apps/studio/state/storage-explorer.tsx"
known_unknowns:
  - "Severity assessments are signal-based, not production-impact based"
  - "Only ~228 test files exist for ~3382 TS/TSX source files, but many features are covered by E2E tests in e2e/studio/"
tags:
  - risk-catalog
  - studio
aliases: []
relates_to:
  - type: "parent"
    target: "[[SYS-STUDIO]]"
  - type: "refines"
    target: "[[PROC-STUDIO-ERROR-HANDLING]]"
sources:
  - category: implementation
    type: github
    ref: "apps/studio/components/interfaces/Auth/AuthProvidersFormValidation.tsx"
    notes: "9 TODO markers in auth provider validation"
  - category: implementation
    type: github
    ref: "apps/studio/components/to-be-cleaned/Table.tsx"
    notes: "6 legacy components in to-be-cleaned/ directory explicitly marked for cleanup"
  - category: implementation
    type: github
    ref: "apps/studio/data/analytics/infra-monitoring-queries.ts"
    notes: "Multiple TODOs for API response format migration"
  - category: implementation
    type: github
    ref: "apps/studio/data/fetchers.ts"
    notes: "Temporary API_URL replace workaround noted by developer"
  - category: implementation
    type: github
    ref: "apps/studio/data/permissions/permissions-query.ts"
    notes: "Untyped API response force-cast"
  - category: implementation
    type: github
    ref: "apps/studio/hooks/ui/useFlag.ts"
    notes: "Two TODOs for refactoring feature flag hook"
  - category: implementation
    type: github
    ref: "apps/studio/lib/sql-parameters.ts"
    notes: "Missing test coverage noted by developer"
  - category: implementation
    type: github
    ref: "apps/studio/state/storage-explorer.tsx"
    notes: "3 TODOs about cache invalidation on rename/move"
notes: ""
---

## Description

Catalogs risk signals found in the Studio codebase during deep scanning. With ~88 TODO/FIXME/HACK signals in components alone plus additional signals in data, lib, state, and hooks layers, the codebase has moderate signal density for its size (~3382 TS/TSX files).

## Key Facts

- 9 TODOs in `AuthProvidersFormValidation.tsx` focus on documentation updates and one unclear property verification (`[TODO] verify what this is?`), indicating provider config forms were built ahead of docs → `apps/studio/components/interfaces/Auth/AuthProvidersFormValidation.tsx`
- The `to-be-cleaned/` directory contains 6 legacy components (KeyMap, AlphaPreview, ListIcons, AutoTextArea, ProductEmptyState, Table) explicitly marked for refactoring → `apps/studio/components/to-be-cleaned/`
- Storage explorer state has 3 TODOs about cache invalidation: rename folders, move files, and rename files all note they should invalidate preview cache but don't → `apps/studio/state/storage-explorer.tsx`
- Infrastructure monitoring queries have 4 TODOs (with matching tests) for removing legacy single-attribute API response format handling once the API always returns multi-attribute format → `apps/studio/data/analytics/infra-monitoring-queries.ts`
- The `API_URL` in fetchers has a temporary `.replace('/platform', '')` workaround noted by developer Joshen, pending env var update to use the base URL directly → `apps/studio/data/fetchers.ts`
- Permissions query response is force-cast (`data as unknown as PermissionsResponse`) with a TODO to type it properly from the API → `apps/studio/data/permissions/permissions-query.ts`
- `useFlag` hook has two TODOs: one to move it to `packages/common/feature-flags.tsx` and another to refactor for explicit loading/disabled/value states → `apps/studio/hooks/ui/useFlag.ts`
- `sql-parameters.ts` has a developer note requesting tests (`[Joshen TODO] We'll want tests for this to ensure that this runs properly`) — no tests exist for this module → `apps/studio/lib/sql-parameters.ts`
- GitHub authorization query has a FIXME noting the retry behavior should be a single check, not retried (`FIXME(kamil): Do not retry, a single check is fine`) → `apps/studio/data/integrations/github-authorization-query.ts`
- Test coverage ratio is ~228 test files to ~3382 source files (~6.7%), though E2E tests in `e2e/studio/` provide additional coverage not counted here → `apps/studio/`
- The error classification system (`error-patterns.ts`) currently has only 1 pattern (ConnectionTimeout); as the system grows, more patterns need to be added to avoid all errors falling into the `UnknownAPIResponseError` bucket → `apps/studio/data/error-patterns.ts`
- Bucket object delete mutation has a TODO about which queries to invalidate after deletion → `apps/studio/data/storage/bucket-object-delete-mutation.ts`

## Findings

| # | Location | Signal | Theme | Severity |
|---|----------|--------|-------|----------|
| 1 | `components/interfaces/Auth/AuthProvidersFormValidation.tsx` | 9 TODOs — docs updates and unclear property | Auth validation debt | medium |
| 2 | `components/to-be-cleaned/` | 6 legacy components pending cleanup | UI debt | medium |
| 3 | `state/storage-explorer.tsx` | 3 TODOs — missing preview cache invalidation | Data consistency | medium |
| 4 | `data/analytics/infra-monitoring-*` | 4 TODOs — legacy API format removal | API migration | low |
| 5 | `data/fetchers.ts` | Temporary API_URL replace workaround | Config debt | low |
| 6 | `data/permissions/permissions-query.ts` | Force-cast untyped API response | Type safety | medium |
| 7 | `hooks/ui/useFlag.ts` | 2 TODOs — needs refactor and relocation | Feature flag debt | low |
| 8 | `lib/sql-parameters.ts` | Missing tests noted by developer | Test coverage | medium |
| 9 | `data/integrations/github-authorization-query.ts` | FIXME — should not retry | Correctness | low |
| 10 | `data/storage/bucket-object-delete-mutation.ts` | TODO — unclear query invalidation | Cache correctness | low |

## Themes

### Auth Provider Form Validation (9 signals, medium)

The auth provider configuration form has the highest concentration of TODOs. Most are documentation-update markers, but one (`[TODO] verify what this is?`) suggests an unclear property whose purpose was not fully understood during implementation.

**Impact if unaddressed:** Dashboard users may encounter confusing or incorrect documentation links when configuring auth providers. The unverified property could produce unexpected behavior.

### Storage Explorer Cache Consistency (3 signals, medium)

Rename and move operations in the storage explorer do not invalidate the file preview cache. Users who rename or move files may see stale previews until the cache naturally expires or they refresh.

**Impact if unaddressed:** Stale file previews after rename/move operations create confusion about whether the operation succeeded.

### Type Safety Gaps (multiple signals, medium)

The permissions query force-casts its response, and the error patterns system has only 1 classified error type. These gaps mean TypeScript cannot catch shape mismatches at compile time.

**Impact if unaddressed:** Runtime errors from API response shape changes would not be caught during development.

### Acknowledged UI Debt (6 components, medium)

The `to-be-cleaned/` directory explicitly marks components for refactoring: KeyMap, AlphaPreview, ListIcons, AutoTextArea, ProductEmptyState, and Table.

**Impact if unaddressed:** Growing technical debt in UI components affects maintainability and consistency.

## Related

- [[SYS-STUDIO]] — parent system
- [[PROC-STUDIO-ERROR-HANDLING]] — error handling gaps are a subset of these risks
