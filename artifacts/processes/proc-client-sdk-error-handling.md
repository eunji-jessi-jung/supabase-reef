---
id: "PROC-CLIENT-SDK-ERROR-HANDLING"
type: "process"
title: "Client SDK Error Handling Across Sub-Clients"
domain: "supabase-reef"
status: draft
last_verified: 2026-04-15
freshness_note: "Deep scan of error classes, fetch handlers, and response processing across all 5 sub-client packages"
freshness_triggers:
  - "packages/core/auth-js/src/lib/errors.ts"
  - "packages/core/auth-js/src/lib/fetch.ts"
  - "packages/core/functions-js/src/types.ts"
  - "packages/core/postgrest-js/src/PostgrestBuilder.ts"
  - "packages/core/storage-js/src/lib/common/errors.ts"
  - "packages/core/supabase-js/src/SupabaseClient.ts"
known_unknowns:
  - "Whether postgrest-js retry behavior (503, 520) is configurable by the end user or hardcoded"
  - "Full catalog of StorageError namespace variants beyond 'storage' and 'vectors'"
tags:
  - client-sdk
  - error-handling
aliases:
  - "SDK Error Handling"
relates_to:
  - type: "parent"
    target: "[[SYS-CLIENT-SDK]]"
  - type: "refines"
    target: "[[PROC-CLIENT-SDK-AUTH]]"
sources:
  - category: implementation
    type: github
    ref: "packages/core/auth-js/src/lib/errors.ts"
    notes: "12 error classes in auth-js error hierarchy"
  - category: implementation
    type: github
    ref: "packages/core/auth-js/src/lib/fetch.ts"
    notes: "Auth fetch with NETWORK_ERROR_CODES handling and API version parsing"
  - category: implementation
    type: github
    ref: "packages/core/auth-js/src/lib/locks.ts"
    notes: "Lock timeout errors: NavigatorLockAcquireTimeoutError, ProcessLockAcquireTimeoutError"
  - category: implementation
    type: github
    ref: "packages/core/functions-js/src/types.ts"
    notes: "FunctionsError hierarchy: FetchError, RelayError, HttpError"
  - category: implementation
    type: github
    ref: "packages/core/postgrest-js/src/PostgrestBuilder.ts"
    notes: "PostgREST error handling with retry logic, AbortError hints, and HeadersOverflow detection"
  - category: implementation
    type: github
    ref: "packages/core/storage-js/src/lib/common/errors.ts"
    notes: "StorageError with namespace support, StorageApiError, StorageUnknownError"
notes: ""
---

## Purpose

Documents how the Client SDK handles and surfaces errors from all sub-clients (Auth, PostgREST, Storage, Functions, Realtime). Each sub-client has its own error class hierarchy but follows a common pattern: typed errors with `{ data, error }` return signatures.

## Key Facts

- **Auth-js** has the richest error hierarchy with 12 error classes: `AuthError` (base), `AuthApiError` (HTTP errors from GoTrue), `AuthRetryableFetchError` (network/5xx), `AuthUnknownError`, `AuthSessionMissingError`, `AuthInvalidTokenResponseError`, `AuthInvalidCredentialsError`, `AuthImplicitGrantRedirectError`, `AuthPKCEGrantCodeExchangeError`, `AuthPKCECodeVerifierMissingError`, `AuthWeakPasswordError`, and `AuthInvalidJwtError` → `packages/core/auth-js/src/lib/errors.ts`
- Auth errors carry `code` (error code string), `status` (HTTP status number), and `__isAuthError` flag; the `isAuthError()` type guard checks for this flag, not `instanceof`, enabling cross-realm detection → `packages/core/auth-js/src/lib/errors.ts`
- Auth fetch treats HTTP status codes 502, 503, 504, 520-524, 530 as `AuthRetryableFetchError` (infrastructure/Cloudflare errors); non-retryable fetch failures also become `AuthRetryableFetchError` with status 0 → `packages/core/auth-js/src/lib/fetch.ts`
- Auth-js special-cases `session_not_found` error code (GoTrue API version 2024-01-01+) into `AuthSessionMissingError`, signaling the session was terminated server-side → `packages/core/auth-js/src/lib/fetch.ts`
- **PostgREST** uses `throwOnError` opt-in pattern: by default, errors are returned as `{ data: null, error: PostgrestError }` objects; calling `.throwOnError()` makes the builder reject the promise instead → `packages/core/postgrest-js/src/PostgrestBuilder.ts`
- PostgREST retries on 503 and 520 status codes with exponential backoff and `Retry-After` header support; fetch-level network errors are also retried → `packages/core/postgrest-js/src/PostgrestBuilder.ts`
- PostgREST provides rich error context for client-side failures: `AbortError` gets a hint about timeout/cancellation with URL length warnings; `HeadersOverflowError` (undici) gets suggestions to use views or RPC functions → `packages/core/postgrest-js/src/PostgrestBuilder.ts`
- **Functions-js** has 3 error subclasses of `FunctionsError`: `FunctionsFetchError` (network failure), `FunctionsRelayError` (Supabase relay cannot reach function), and `FunctionsHttpError` (non-2xx response) → `packages/core/functions-js/src/types.ts`
- **Storage-js** has `StorageError` base with namespace support ('storage' or 'vectors'), `StorageApiError` (server errors with status/statusCode), and `StorageUnknownError` (wraps unexpected errors with `originalError` property) → `packages/core/storage-js/src/lib/common/errors.ts`
- Lock acquisition errors (`NavigatorLockAcquireTimeoutError`, `ProcessLockAcquireTimeoutError`) extend `LockAcquireTimeoutError` with an `isAcquireTimeout` boolean flag for type-safe detection without `instanceof` → `packages/core/auth-js/src/lib/locks.ts`
- All error base classes include `toJSON()` methods for serialization — `AuthError`, `FunctionsError`, and specialized subclasses serialize cleanly for logging and Sentry → `packages/core/auth-js/src/lib/errors.ts` and `packages/core/functions-js/src/types.ts`

## Steps

1. **Request initiation** — Sub-client makes HTTP request via the shared `fetchWithAuth` wrapper (for PostgREST, Storage, Functions) or direct fetch (for Auth)
2. **Network error handling** — If `fetch()` throws (network failure, CORS), sub-clients wrap it in their typed error: `AuthRetryableFetchError`, `FunctionsFetchError`, or PostgREST's `{ error: { message, details, hint } }` object
3. **HTTP error classification** — Non-OK responses are parsed: auth-js checks error codes and status ranges; PostgREST checks for retry eligibility (503, 520); Functions distinguishes relay vs HTTP errors
4. **Retry logic** — Auth and PostgREST retry on specific status codes; auth retries up to 10 times at 200ms intervals; PostgREST uses exponential backoff with `Retry-After` support
5. **Error surfacing** — Errors are returned as `{ data: null, error }` (default) or thrown as exceptions (opt-in via `throwOnError` in PostgREST, `throwOnError` option in auth)
6. **Consumer handling** — Application code checks `if (error)` on the response object, or uses `try/catch` if `throwOnError` was enabled

## Worked Examples

### PostgREST Error with Helpful Hints

A developer's query generates a URL over 8000 characters using `.in('id', [200+ UUIDs])`. The request fails with `AbortError` because the URL exceeds server limits. PostgREST's error handler detects the URL length exceeds `urlLengthLimit`, and produces `{ error: { message: "AbortError: ...", hint: "Request was aborted... Note: Your request URL is 8234 characters, which may exceed server limits. If filtering with large arrays, consider using an RPC function..." } }`.

### Auth Token Refresh Failure

During auto-refresh, the GoTrue endpoint returns a 401 with error code `invalid_grant`. Auth-js `handleError` creates an `AuthApiError` with status 401 and code `invalid_grant`. Since this is not retryable (`isAuthRetryableFetchError` returns false), GoTrueClient's `_autoRefreshTokenTick` calls `_removeSession()`, clearing the stored session and firing `SIGNED_OUT`.

## Related

- [[SYS-CLIENT-SDK]] — parent system
- [[PROC-CLIENT-SDK-AUTH]] — auth errors are the most complex subset of SDK error handling
