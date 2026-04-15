---
id: "RISK-STORAGE-KNOWN-GAPS"
type: "risk"
title: "Storage Known Risks, Gaps, and Backlog Candidates"
domain: "supabase-reef"
status: draft
last_verified: 2026-04-15
freshness_note: "Deep scan of TODO/FIXME signals, error handler duplication, Knex patch, multi-tenant isolation patterns, and protocol-specific gaps."
freshness_triggers:
  - "patches/knex+3.1.0.patch"
  - "src/http/error-handler.ts"
  - "src/http/routes/object/getSignedUploadURL.ts"
  - "src/http/routes/s3/error-handler.ts"
  - "src/internal/database/connection.ts"
  - "src/storage/database/knex.ts"
  - "src/storage/protocols/s3/s3-handler.ts"
known_unknowns:
  - "Whether the Knex patch has been proposed upstream or will diverge further"
  - "Whether RLS policies have gaps not visible from application code alone"
  - "Production impact of the S3 error handler missing the Supavisor auth error pattern"
tags:
  - risk-catalog
  - storage
  - security
  - multi-tenant
aliases: []
relates_to:
  - type: "parent"
    target: "[[SYS-STORAGE]]"
  - type: "refines"
    target: "[[PROC-STORAGE-ERROR-HANDLING]]"
sources:
  - category: implementation
    type: github
    ref: "patches/knex+3.1.0.patch"
    notes: "Custom Knex patch for PgBouncer-compatible query cancellation and AbortSignal support"
  - category: implementation
    type: github
    ref: "src/http/error-handler.ts"
    notes: "Global error handler with 6 connection-limit patterns"
  - category: implementation
    type: github
    ref: "src/http/routes/object/getSignedUploadURL.ts"
    notes: "TODO about unvalidated user metadata types"
  - category: implementation
    type: github
    ref: "src/http/routes/s3/error-handler.ts"
    notes: "S3 error handler with 5 connection-limit patterns (missing one)"
  - category: implementation
    type: github
    ref: "src/internal/database/connection.ts"
    notes: "TenantConnection scope-setting and transaction retry logic"
  - category: implementation
    type: github
    ref: "src/internal/database/tenant.ts"
    notes: "Tenant config caching, JWKS merging, PubSub invalidation"
  - category: implementation
    type: github
    ref: "src/storage/database/knex.ts"
    notes: "Two TODOs about migration-aware column normalization"
  - category: implementation
    type: github
    ref: "src/storage/protocols/s3/s3-handler.ts"
    notes: "TODO for unimplemented S3 tagging"
notes: ""
---

## Description

Catalogs risk signals, code gaps, and architectural concerns found in the Storage codebase during deep-dive analysis. Signals span TODO markers, duplicated logic, a substantial Knex patch, multi-tenant isolation patterns, and protocol-specific incompleteness.

## Key Facts

- S3 error handler checks only 5 database connection-limit message patterns while the global REST handler checks 6 (the S3 handler is missing "Authentication error" from Supavisor), meaning S3 clients may get a 500 instead of 429 during Supavisor auth failures → `src/http/routes/s3/error-handler.ts`
- The Knex 3.1.0 patch is 288 lines across 5 files, adding `abortOnSignal()` to QueryBuilder and Raw, replacing `pg_cancel_backend` with native PostgreSQL cancel protocol for PgBouncer compatibility, and adding abort signal wiring to the Runner -- this is a significant fork surface → `patches/knex+3.1.0.patch`
- Two TODOs in the Knex database adapter guard against a missing `metadata` column on `s3_multipart_uploads` based on migration version checks that should be moved into `normalizeColumns` once it becomes table-aware → `src/storage/database/knex.ts`
- S3 `getObjectTagging` returns a hardcoded empty `TagSet` with a `TODO: implement tagging when supported` comment, meaning S3 clients using object tagging will get silent empty results → `src/storage/protocols/s3/s3-handler.ts`
- `parseUserMetadata` in signed upload URL handler casts values to `Record<string, string>` without validation; the TODO acknowledges values could be any type → `src/http/routes/object/getSignedUploadURL.ts`
- Tenant config (including JWT secrets) is cached in an LRU cache with 1-hour TTL and invalidated via PubSub; if PubSub delivery fails, stale secrets could persist for up to 1 hour → `src/internal/database/tenant.ts`
- `TenantConnection.setScope()` injects all RLS context via a single `SELECT set_config(...)` call with 10 parameters; if any parameter is corrupted or truncated, the entire RLS context could be wrong for that transaction → `src/internal/database/connection.ts`
- The REST error handler collapses most non-500 errors to HTTP 400 by default (`userStatusCode` logic), which may confuse clients expecting semantic HTTP status codes (e.g., 404 for not found, 409 for conflict) → `src/http/error-handler.ts`
- Transaction creation retries up to 10 times (50-200ms backoff, 3s max) only for connection-limit errors; other transient database errors (e.g., serialization failures) are not retried → `src/internal/database/connection.ts`
- The Knex patch marks connections as `__knex__disposed` after abort signal cancellation, preventing pool reuse, but the timing between abort and query completion could leave connections in an uncertain state → `patches/knex+3.1.0.patch`

## Findings

| # | Location | Signal | Theme | Severity |
|---|----------|--------|-------|----------|
| 1 | `src/http/routes/s3/error-handler.ts` | S3 handler missing Supavisor "Authentication error" pattern | Error handling duplication | medium |
| 2 | `patches/knex+3.1.0.patch` | 288-line patch adding abort signal and PgBouncer cancel protocol | Dependency fork | high |
| 3 | `src/storage/database/knex.ts` | 2x TODO: move migration guard into normalizeColumns | Database adapter | medium |
| 4 | `src/storage/protocols/s3/s3-handler.ts` | TODO: implement tagging when supported | S3 protocol completeness | low |
| 5 | `src/http/routes/object/getSignedUploadURL.ts` | TODO: validate user metadata types | Input validation | medium |
| 6 | `src/internal/database/tenant.ts` | Tenant config cached 1h, PubSub invalidation not guaranteed | Multi-tenant isolation | medium |
| 7 | `src/internal/database/connection.ts` | setScope injects 10 params in single call, no validation | RLS context integrity | medium |
| 8 | `src/http/error-handler.ts` | Non-500 errors collapsed to 400 by default | API semantics | low |
| 9 | `src/internal/database/connection.ts` | Transaction retry only for connection-limit errors | Resilience | low |
| 10 | `patches/knex+3.1.0.patch` | Abort cancellation timing vs pool reuse | Connection safety | medium |

## Themes

### Knex Patch Maintenance (2 signals, high)

The 288-line Knex 3.1.0 patch touches 5 files to add PgBouncer-compatible query cancellation and AbortSignal support. This represents a significant divergence from upstream Knex that must be maintained across upgrades. The patch adds `abortOnSignal()` to both `QueryBuilder` and `Raw`, replaces `pg_cancel_backend` with native cancel protocol, and introduces connection disposal on abort.

**Impact if unaddressed:** Knex upgrades become risky and labor-intensive; the abort/cancellation behavior diverges further from upstream.

### Error Handler Duplication (1 signal, medium)

The S3 error handler duplicates the database connection-limit detection logic from the global handler but with fewer patterns (5 vs 6). The missing "Authentication error" pattern from Supavisor means S3 clients could receive a 500 Internal Server Error instead of the expected 429 SlowDown during Supavisor authentication failures.

**Impact if unaddressed:** Inconsistent error responses between REST and S3 protocols; S3 clients may not implement appropriate retry logic for connection exhaustion.

### Multi-tenant Isolation (2 signals, medium)

Tenant config caching with 1-hour TTL relies on PubSub for invalidation. If PubSub delivery fails (network partition, subscriber restart), a revoked JWT secret could remain valid for up to 1 hour. Additionally, the RLS scope-setting call injects 10 parameters without input validation on the application side.

**Impact if unaddressed:** Potential window where revoked credentials remain usable; corrupted scope parameters could weaken RLS enforcement for individual requests.

### Protocol Completeness (1 signal, low)

S3 object tagging is stub-implemented, returning empty TagSet. Clients relying on S3 tagging for lifecycle policies or access control will silently get no results.

**Impact if unaddressed:** Silent data loss for workflows depending on S3 object tags.

### Input Validation (1 signal, medium)

User metadata parsing in signed upload URLs casts values to `Record<string, string>` without type validation, as noted in the TODO. Non-string values could cause unexpected behavior downstream.

**Impact if unaddressed:** Potential for type confusion or injection in user metadata fields.

## Related

- [[SYS-STORAGE]] -- parent system
- [[PROC-STORAGE-ERROR-HANDLING]] -- details the error handling architecture where duplication was found
