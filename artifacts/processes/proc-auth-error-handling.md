---
id: "PROC-AUTH-ERROR-HANDLING"
type: "process"
title: "Auth Error Handling Patterns"
domain: "supabase-reef"
status: draft
last_verified: 2026-04-15
freshness_note: "Deep scan of apierrors package, errors.go, hooks error handling, and external OAuth error redirect flow"
freshness_triggers:
  - "internal/api/apierrors/"
  - "internal/api/errors.go"
  - "internal/api/external.go"
  - "internal/hooks/hookserrors/"
known_unknowns:
  - "Whether PostgresError mapping covers all relevant Postgres error codes"
  - "How error_id propagation works end-to-end from server logs to client debugging"
tags:
  - error-handling
  - auth
aliases: []
relates_to:
  - type: "parent"
    target: "[[SYS-AUTH]]"
  - type: "refines"
    target: "[[PROC-AUTH-AUTH]]"
sources:
  - category: "implementation"
    type: "github"
    ref: "internal/api/apierrors/apierrors.go"
  - category: "implementation"
    type: "github"
    ref: "internal/api/apierrors/errorcode.go"
  - category: "implementation"
    type: "github"
    ref: "internal/api/errors.go"
  - category: "implementation"
    type: "github"
    ref: "internal/api/external.go"
  - category: "implementation"
    type: "github"
    ref: "internal/hooks/hookserrors/hookserrors.go"
notes: ""
---

## Purpose

Documents how Auth handles errors across its API surface: the type hierarchy, HTTP response formatting, version-aware error shapes, OAuth redirect error encoding, hook error propagation, and panic recovery.

## Key Facts

- Two primary error types: `HTTPError` (structured API errors with HTTP status, error code, and message) and `OAuthError` (OAuth2 spec errors with `error` and `error_description`) → `internal/api/apierrors/apierrors.go`
- Both error types support fluent chaining: `.WithInternalError(err)` for root cause and `.WithInternalMessage(fmt, args)` for internal-only messages hidden from clients → `internal/api/apierrors/apierrors.go`
- 90+ typed error codes defined as constants in `ErrorCode` type alias covering auth, MFA, SAML, SSO, OAuth, passkeys, rate limits, hooks, and web3 → `internal/api/apierrors/errorcode.go`
- Convenience constructors for common HTTP statuses: `NewBadRequestError`, `NewForbiddenError`, `NewNotFoundError`, `NewUnprocessableEntityError`, `NewTooManyRequestsError`, `NewInternalServerError`, `NewConflictError` → `internal/api/apierrors/apierrors.go`
- `HandleResponseError` is the central error handler that dispatches on error type (`WeakPasswordError`, `HTTPError`, `OAuthError`, `ErrorCause`, or default) with version-aware formatting → `internal/api/errors.go`
- API version `20240101` returns simplified JSON shape `{code, message}` instead of legacy `{code, error_code, msg}` with HTTP status in body; version determined from `X-Supabase-Api-Version` header → `internal/api/errors.go`
- 5xx errors get an `error_id` (request ID) injected and are logged at error level with stack trace; 429s logged at warn; all others at info → `internal/api/errors.go`
- Error code is echoed in `x-sb-error-code` response header for machine-readable error identification → `internal/api/errors.go`
- Postgres errors are detected and translated to user-friendly messages with appropriate HTTP status codes via `utilities.NewPostgresError`, bypassing the normal HTTPError response → `internal/api/errors.go`
- `recoverer` middleware catches panics and returns a 500 HTTPError, logging panic details and stack trace → `internal/api/errors.go`
- OAuth redirect errors are encoded as both query parameters and URL fragment on the redirect URL, with an `sb` marker to identify Supabase Auth redirects → `internal/api/external.go`
- OAuth error mapping translates HTTP status codes to OAuth2 error strings: 400=invalid_request, 401=unauthorized_client, 403=access_denied, 500=server_error, 503=temporarily_unavailable → `internal/api/errors.go`
- Hook errors use a separate `hookserrors.Error` type with `http_code` and `message` fields; defaults to 500 when http_code is 0 (acknowledged as needing to be 400) → `internal/hooks/hookserrors/hookserrors.go`
- Hook error detection checks for non-empty `message` field in the `error` JSON object; an error with `http_code=500` but empty message is silently ignored → `internal/hooks/hookserrors/hookserrors.go`
- Unhandled errors (not matching any known type) produce a generic 500 with `unexpected_failure` code and message directing to server logs → `internal/api/errors.go`

## Steps

1. **Error creation** -- handler or middleware creates `HTTPError` or `OAuthError` with typed error code and message
2. **Error propagation** -- error bubbles up through handler chain; `ErrorCause` interface allows unwrapping nested errors
3. **Central dispatch** -- `HandleResponseError` receives the error, determines API version from request header
4. **Type switching** -- dispatches to appropriate formatting: WeakPasswordError (with reasons), HTTPError (with version-aware shape), OAuthError (always 400), ErrorCause (recursive unwrap), default (generic 500)
5. **Logging** -- 5xx errors logged at error level with request ID and stack trace; 429 at warn; others at info
6. **Response** -- JSON response sent with appropriate HTTP status; error code echoed in `x-sb-error-code` header

## Related

- [[SYS-AUTH]] -- parent system
- [[PROC-AUTH-AUTH]] -- authentication flow that produces these errors
