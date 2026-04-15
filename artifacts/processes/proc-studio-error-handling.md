---
id: "PROC-STUDIO-ERROR-HANDLING"
type: "process"
title: "Studio Error Handling and Display Pipeline"
domain: "supabase-reef"
status: draft
last_verified: 2026-04-15
freshness_note: "Deep scan of error boundaries, fetchers, error-patterns, ErrorMatcher, AlertError, and API error types"
freshness_triggers:
  - "apps/studio/components/interfaces/ErrorHandling/"
  - "apps/studio/components/ui/AlertError.tsx"
  - "apps/studio/components/ui/Error.tsx"
  - "apps/studio/components/ui/ErrorBoundary/"
  - "apps/studio/data/error-patterns.ts"
  - "apps/studio/data/fetchers.ts"
  - "apps/studio/types/api-errors.ts"
  - "apps/studio/types/base.ts"
known_unknowns:
  - "How many error patterns will be added beyond ConnectionTimeoutError — the system is extensible but currently has only one mapping"
  - "Whether toast notifications are used consistently or some error paths bypass them"
  - "Full catalog of Sentry context fields attached to captured exceptions"
tags:
  - studio
  - error-handling
  - sentry
  - error-boundaries
aliases:
  - "Studio Error Pipeline"
relates_to:
  - type: "parent"
    target: "[[SYS-STUDIO]]"
  - type: "depends_on"
    target: "[[PROC-STUDIO-AUTH]]"
sources:
  - category: implementation
    type: github
    ref: "apps/studio/components/interfaces/ErrorHandling/ErrorMatcher.tsx"
    notes: "Central error display component with telemetry tracking"
  - category: implementation
    type: github
    ref: "apps/studio/components/interfaces/ErrorHandling/ErrorMatcher.utils.ts"
    notes: "Maps ResponseError instances to ErrorMapping via instanceof checks"
  - category: implementation
    type: github
    ref: "apps/studio/components/interfaces/ErrorHandling/error-mappings.tsx"
    notes: "Registry of classified errors to troubleshooting components"
  - category: implementation
    type: github
    ref: "apps/studio/components/interfaces/ErrorHandling/errorMappings/ConnectionTimeout.tsx"
    notes: "Inline troubleshooting with restart, guide, and AI debug steps"
  - category: implementation
    type: github
    ref: "apps/studio/components/ui/AlertError.tsx"
    notes: "Standard error admonition with support link and telemetry"
  - category: implementation
    type: github
    ref: "apps/studio/components/ui/Error.tsx"
    notes: "Full-page error fallback component"
  - category: implementation
    type: github
    ref: "apps/studio/components/ui/ErrorBoundary/ErrorBoundary.tsx"
    notes: "React error boundary wrapper with Sentry reporting"
  - category: implementation
    type: github
    ref: "apps/studio/components/ui/ErrorBoundary/GlobalErrorBoundaryState.tsx"
    notes: "Global app-level error boundary with special DOM error handling"
  - category: implementation
    type: github
    ref: "apps/studio/data/error-patterns.ts"
    notes: "Regex-to-ErrorClass mapping for API response classification"
  - category: implementation
    type: github
    ref: "apps/studio/data/fetchers.ts"
    notes: "handleError function that classifies and throws typed errors"
  - category: implementation
    type: github
    ref: "apps/studio/types/api-errors.ts"
    notes: "ConnectionTimeoutError and UnknownAPIResponseError class definitions"
  - category: implementation
    type: github
    ref: "apps/studio/types/base.ts"
    notes: "ResponseError base class with code, requestId, retryAfter, metadata"
notes: ""
---

## Purpose

Documents the multi-layered error handling pipeline in Studio, from API response classification through error boundaries to user-facing display with inline troubleshooting.

## Key Facts

- `ResponseError` is the base error class carrying `code`, `requestId`, `retryAfter`, `requestPathname`, and optional `metadata` (e.g., SQL cost) — all API errors inherit from it → `apps/studio/types/base.ts`
- `handleError` in `fetchers.ts` is the central error handler: it matches error messages against `ERROR_PATTERNS` regex list and throws the matching typed error subclass, or falls back to `UnknownAPIResponseError` → `apps/studio/data/fetchers.ts`
- Error classification uses a `Map<ErrorConstructor, RegExp>` pattern, currently mapping `ConnectionTimeoutError` to the regex `/CONNECTION\s+TERMINATED\s+DUE\s+TO\s+CONNECTION\s+TIMEOUT/i` → `apps/studio/data/error-patterns.ts`
- Two classified error types exist: `ConnectionTimeoutError` (errorType: 'connection-timeout') and `UnknownAPIResponseError` (errorType: 'unknown'); both extend `ResponseError` → `apps/studio/types/api-errors.ts`
- The HTTP middleware enriches error responses with `code` (HTTP status), `requestId`, `retryAfter`, and `requestPathname` before they reach `handleError` → `apps/studio/data/fetchers.ts`
- `ErrorMatcher` is the primary UI component for displaying classified errors; it looks up a `Troubleshooting` component via `getMappingForError()` and tracks `dashboard_error_created` and `inline_error_troubleshooter_exposed` telemetry events → `apps/studio/components/interfaces/ErrorHandling/ErrorMatcher.tsx`
- `getMappingForError` only matches `ResponseError` instances (via `instanceof` check) against the `ERROR_MAPPINGS` Map; non-ResponseError objects return null → `apps/studio/components/interfaces/ErrorHandling/ErrorMatcher.utils.ts`
- `ConnectionTimeoutTroubleshooting` provides 3-step inline troubleshooting: restart database, follow troubleshooting guide, and debug with AI assistant — the AI button opens the sidebar with a pre-built prompt → `apps/studio/components/interfaces/ErrorHandling/errorMappings/ConnectionTimeout.tsx`
- `AlertError` is a generic error admonition component using `Admonition` from `ui-patterns`; it includes a "Contact support" link pre-filled with error details and tracks `dashboard_error_created` at 10% sample rate → `apps/studio/components/ui/AlertError.tsx`
- `ErrorBoundary` wraps React subtrees with `react-error-boundary`; on error, it captures the exception to Sentry with `componentStack` and optional `sentryContext`, then shows a "Try again" button with optional custom actions → `apps/studio/components/ui/ErrorBoundary/ErrorBoundary.tsx`
- `GlobalErrorBoundaryState` handles app-level crashes with special cases for DOM mutation errors (`removeChild`/`insertBefore` on Node), which are typically caused by browser extensions; it displays Sentry issue ID when available → `apps/studio/components/ui/ErrorBoundary/GlobalErrorBoundaryState.tsx`
- The full-page `Error` component (`Error.tsx`) is a simpler fallback showing error code and message with "Head back" and "Submit a support request" buttons → `apps/studio/components/ui/Error.tsx`
- `fetchHandler` wraps native `fetch` to convert `TypeError: Failed to fetch` into a user-friendly "Unable to reach the server" message → `apps/studio/data/fetchers.ts`
- Unrecognized errors (no message/msg property) are captured to Sentry and re-thrown as `UnknownAPIResponseError` with an intentionally vague message to avoid leaking internals to the UI → `apps/studio/data/fetchers.ts`

## Steps

1. **API call** — Studio makes HTTP request via `openapi-fetch` client with auth middleware
2. **Response middleware** — Non-OK responses are enriched with `code`, `requestId`, `retryAfter`, `requestPathname` in the `onResponse` middleware
3. **Error classification** — `handleError` matches error message against `ERROR_PATTERNS` regexes; throws typed subclass (`ConnectionTimeoutError`) or `UnknownAPIResponseError`
4. **React Query capture** — The thrown error propagates to React Query, which stores it as the query/mutation error state
5. **UI display** — Components use `ErrorMatcher` (for classified errors with troubleshooting), `AlertError` (for generic warning admonitions), or the global `ErrorBoundary` (for unhandled exceptions)
6. **Telemetry** — Error display components track `dashboard_error_created` events; classified errors additionally track `inline_error_troubleshooter_exposed`
7. **Sentry reporting** — `ErrorBoundary` captures exceptions with component stack; `handleError` captures unrecognized errors; both attach context metadata

## Worked Examples

### Connection Timeout Error Flow

A database query in Studio times out. PostgREST returns an error with message containing "CONNECTION TERMINATED DUE TO CONNECTION TIMEOUT". The `onResponse` middleware enriches it with status code and request ID. `handleError` matches the regex from `ERROR_PATTERNS`, throws `ConnectionTimeoutError`. The React Query hook surfaces this error. The `ErrorMatcher` component finds the `ConnectionTimeoutTroubleshooting` mapping and renders a 3-step accordion: restart project, follow docs guide, debug with AI.

### Unclassified API Error Flow

A permissions API call fails with a 500 error and message "Internal server error". `handleError` finds no regex match in `ERROR_PATTERNS`. It throws `UnknownAPIResponseError`. In the UI, `ErrorMatcher` calls `getMappingForError()` which returns `null` (no troubleshooting available). The error displays with the raw message and a generic support link.

## Related

- [[SYS-STUDIO]] — parent system
- [[PROC-STUDIO-AUTH]] — auth errors are a subset of the error pipeline
