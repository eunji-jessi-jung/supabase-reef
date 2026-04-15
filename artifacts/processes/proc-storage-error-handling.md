---
id: "PROC-STORAGE-ERROR-HANDLING"
type: "process"
title: "Storage Error Handling Across Protocols"
domain: "supabase-reef"
status: draft
last_verified: 2026-04-15
freshness_note: "Deep read of error class hierarchy, global error handler, S3-specific error handler, TUS error mapping, and error codes enum."
freshness_triggers:
  - "src/http/error-handler.ts"
  - "src/http/routes/s3/error-handler.ts"
  - "src/http/routes/tus/lifecycle.ts"
  - "src/internal/errors/"
known_unknowns:
  - "Whether Iceberg protocol routes have their own error handler or reuse the global one"
  - "How vector route errors are formatted (S3-XML vs JSON)"
tags:
  - storage
  - error-handling
  - fastify
aliases:
  - "Storage Error Flow"
relates_to:
  - type: "parent"
    target: "[[SYS-STORAGE]]"
  - type: "refines"
    target: "[[DEC-MULTI-PROTOCOL-STORAGE]]"
sources:
  - category: implementation
    type: github
    ref: "src/http/error-handler.ts"
    notes: "Global Fastify error handler for REST/HTTP routes"
  - category: implementation
    type: github
    ref: "src/http/routes/s3/error-handler.ts"
    notes: "S3-specific error handler producing XML-shaped Error responses"
  - category: implementation
    type: github
    ref: "src/http/routes/tus/lifecycle.ts"
    notes: "TUS error mapping in onUploadFinish and onResponseError"
  - category: implementation
    type: github
    ref: "src/internal/errors/codes.ts"
    notes: "ErrorCode enum and ERRORS factory, normalizeRawError"
  - category: implementation
    type: github
    ref: "src/internal/errors/renderable.ts"
    notes: "RenderableError interface and StorageError type"
  - category: implementation
    type: github
    ref: "src/internal/errors/render.ts"
    notes: "Generic render() function"
  - category: implementation
    type: github
    ref: "src/internal/errors/storage-error.ts"
    notes: "StorageBackendError class with render(), fromError(), connection close"
notes: ""
---

## Purpose

Describes how Storage handles errors across its multi-protocol surface (REST, S3, TUS). Each protocol has a distinct error response format, but all share a common error class hierarchy rooted in `StorageBackendError`. The system distinguishes between renderable (user-facing) errors and unexpected internal errors, with protocol-specific formatters ensuring clients receive errors in the expected shape.

## Key Facts

- `StorageBackendError` is the central error class; it implements `RenderableError` and carries `code` (ErrorCode enum), `httpStatusCode`, `userStatusCode`, `message`, `originalError`, and optional `resource` and `metadata` → `src/internal/errors/storage-error.ts`
- The `ERRORS` object is a factory of 50+ named constructors (e.g., `ERRORS.AccessDenied()`, `ERRORS.NoSuchKey()`, `ERRORS.DatabaseTimeout()`) that return pre-configured `StorageBackendError` instances with correct HTTP status codes → `src/internal/errors/codes.ts`
- The `ErrorCode` enum defines 50+ domain-specific error codes spanning auth (AccessDenied, InvalidJWT, SignatureDoesNotMatch), database (DatabaseTimeout, DatabaseConnectionLimit, DatabaseReadOnly, DatabaseSchemaMismatch), S3 (S3Error, InvalidAccessKeyId), TUS (TusError), and resource errors (NoSuchBucket, NoSuchKey, EntityTooLarge) → `src/internal/errors/codes.ts`
- The global REST error handler (`setErrorHandler`) intercepts `DatabaseError` connection-limit messages (6 known patterns including Supavisor's "Authentication error") and returns 429 with `SlowDown` code before checking other error types → `src/http/error-handler.ts`
- For renderable errors on REST routes, `userStatusCode` overrides the raw HTTP status -- errors with `httpStatusCode: 500` return 500, all others default to 400 unless `respectStatusCode` is set, allowing the system to hide internal details → `src/http/error-handler.ts`
- `AbortedTerminate` errors trigger `Connection: close` header and schedule socket destruction 3 seconds after response finish, preventing connection reuse on fatal errors → `src/http/error-handler.ts`
- The S3 error handler produces a different response shape: `{ Error: { Resource, Code, Message } }` matching AWS S3 API conventions, with the resource extracted from the URL path → `src/http/routes/s3/error-handler.ts`
- S3 error handler has its own duplicated DatabaseError connection-limit check (5 patterns vs 6 in the global handler -- missing the Supavisor "Authentication error" pattern) → `src/http/routes/s3/error-handler.ts`
- `StorageBackendError.fromError()` wraps unknown errors: AWS S3ServiceException errors get code `S3Error` with the upstream HTTP status, all other errors become `InternalError` with status 500 → `src/internal/errors/storage-error.ts`
- TUS protocol errors are mapped in `onUploadFinish` by patching `status_code` and `body` onto renderable errors, and in `onResponseError` by wrapping non-Error objects as `TusError` with metadata → `src/http/routes/tus/lifecycle.ts`
- The `render()` utility function checks `isRenderableError` first (duck-typing on `.render()` method); unrecognized errors fall through to `StorageBackendError.fromError()` which always produces a renderable error → `src/internal/errors/render.ts`
- `normalizeRawError()` in codes.ts serializes any error to a structured log payload with `raw` (JSON), `name`, `message`, `stack`, and `statusCode`, handling non-Error objects gracefully → `src/internal/errors/codes.ts`
- All errors are attached to `request.executionError` for the log-request plugin to include in structured request logs → `src/http/error-handler.ts`

## Worked Examples

### REST Upload to Non-existent Bucket

1. Request hits `POST /object/:bucketName/*`
2. Storage layer calls `findBucket()` which returns null
3. Code throws `ERRORS.NoSuchBucket(bucketName)` -- creates `StorageBackendError` with code `NoSuchBucket`, httpStatusCode 404
4. Global error handler catches it, calls `.render()` -> `{ statusCode: "404", code: "NoSuchBucket", error: "NoSuchBucket", message: "Bucket not found" }`
5. `userStatusCode` is 400 (non-500 default), so response status is 400 (not 404) unless `respectStatusCode` is enabled

### S3 Signature Mismatch

1. S3 request arrives with invalid AWS Signature V4
2. `authorizeRequestSignV4` calls `signatureV4.verify()` which returns false
3. Code throws `ERRORS.SignatureDoesNotMatch(...)` -- httpStatusCode 403
4. S3 error handler catches `StorageBackendError`, returns `{ Error: { Resource: "bucket/key", Code: "SignatureDoesNotMatch", Message: "..." } }` with status 403

### Database Connection Exhaustion

1. Any request triggers database query
2. `pg` throws `DatabaseError` with message "Max client connections reached"
3. Both global and S3 error handlers have explicit checks for this pattern
4. Response: 429 with code `SlowDown` and message "Too many connections issued to the database"

## Related

- [[SYS-STORAGE]] -- parent system
- [[DEC-MULTI-PROTOCOL-STORAGE]] -- decision record explaining why multiple error formats exist
