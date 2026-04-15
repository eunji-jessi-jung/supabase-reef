---
id: "PAT-CROSS-SERVICE-RETRY"
type: "pattern"
title: "Cross-Service Retry Pattern"
domain: "supabase-reef"
status: draft
last_verified: 2026-04-15
freshness_note: "scuba-depth cross-service pattern analysis"
freshness_triggers:
  - "packages/core/postgrest-js/src/PostgrestBuilder.ts"
  - "packages/core/postgrest-js/src/types/common/common.ts"
  - "packages/core/auth-js/src/lib/fetch.ts"
  - "packages/core/auth-js/src/lib/constants.ts"
  - "packages/core/realtime-js/src/RealtimeClient.ts"
  - "apps/studio/data/query-client.ts"
known_unknowns:
  - "Whether Storage server has any internal retry logic for backend S3 calls"
  - "How Realtime's reconnect intervals interact with channel rejoin logic"
  - "Whether Auth server retries failed hook calls (HTTP/pg-func) or just reports errors"
tags:
  - pattern
  - cross-service
  - retry
  - resilience
aliases:
  - "Retry and Resilience Pattern"
relates_to:
  - type: "related"
    target: "[[PROC-CLIENT-SDK-AUTH]]"
  - type: "related"
    target: "[[PROC-STUDIO-AUTH]]"
  - type: "related"
    target: "[[PAT-CROSS-SERVICE-MIDDLEWARE]]"
  - type: "related"
    target: "[[PAT-CROSS-SERVICE-DATA-FLOW]]"
sources:
  - category: "implementation"
    type: "github"
    ref: "packages/core/postgrest-js/src/PostgrestBuilder.ts"
  - category: "implementation"
    type: "github"
    ref: "packages/core/postgrest-js/src/types/common/common.ts"
  - category: "implementation"
    type: "github"
    ref: "packages/core/auth-js/src/lib/fetch.ts"
  - category: "implementation"
    type: "github"
    ref: "packages/core/auth-js/src/lib/constants.ts"
  - category: "implementation"
    type: "github"
    ref: "packages/core/realtime-js/src/RealtimeClient.ts"
  - category: "implementation"
    type: "github"
    ref: "apps/studio/data/query-client.ts"
notes: ""
---

## Overview

Retry logic appears in two places in the Supabase ecosystem: the Client SDK (postgrest-js, auth-js, realtime-js) and Studio (React Query). Each implements retry differently based on its protocol and error characteristics. PostgREST retries only idempotent methods on specific status codes. Auth-js classifies errors as retryable or non-retryable (with non-retryable errors causing session termination). Realtime uses stepped reconnect intervals for WebSocket failures. Studio uses React Query's built-in retry with custom delay logic that respects Retry-After headers.

## Key Facts

- PostgREST-js retries up to 3 times (`DEFAULT_MAX_RETRIES`) with exponential backoff (1s, 2s, 4s, capped at 30s) → `packages/core/postgrest-js/src/types/common/common.ts`
- PostgREST-js only retries idempotent methods (GET, HEAD, OPTIONS) on status codes 520 (Cloudflare) and 503 (PostgREST schema cache loading) → `packages/core/postgrest-js/src/types/common/common.ts`
- PostgREST-js retry uses an AbortSignal-aware `sleep()` function that resolves early if the request is cancelled → `packages/core/postgrest-js/src/PostgrestBuilder.ts`
- PostgREST-js retry is enabled by default (`retryEnabled = true`) but can be disabled per-builder instance → `packages/core/postgrest-js/src/PostgrestBuilder.ts`
- Auth-js classifies HTTP status codes 502, 503, 504, 520-524, 530 as `NETWORK_ERROR_CODES` that throw `AuthRetryableFetchError` → `packages/core/auth-js/src/lib/fetch.ts`
- Auth-js fetch failures (network/CORS errors) also throw `AuthRetryableFetchError` with status 0, distinguishing network failures from server errors → `packages/core/auth-js/src/lib/fetch.ts`
- Auth-js token refresh uses up to 10 retries with 200ms intervals (`NETWORK_FAILURE.RETRY_INTERVAL_MS`) for transient errors → `packages/core/auth-js/src/lib/constants.ts`
- Auth-js terminates the session entirely if auto-refresh fails with a non-retryable error (e.g., invalid refresh token) → `packages/core/auth-js/src/GoTrueClient.ts`
- Realtime-js uses stepped reconnect intervals: 1s, 2s, 5s, 10s, then 10s (constant) for WebSocket reconnection → `packages/core/realtime-js/src/RealtimeClient.ts`
- Studio's React Query retries up to 3 times (`MAX_RETRY_FAILURE_COUNT`) with exponential backoff (1s, 2s, 4s, capped at 30s) → `apps/studio/data/query-client.ts`
- Studio skips retries on 4xx errors (except 429 rate limits), preventing pointless retries on client errors like 401, 403, 404 → `apps/studio/data/query-client.ts`
- Studio skips retries for specific expensive endpoints (`/run-lints`, `/usage`, `/query`, `/logs.all`) except for 429 errors, to avoid amplifying load → `apps/studio/data/query-client.ts`
- Studio respects `Retry-After` headers from 429 responses, using the server-specified delay instead of exponential backoff → `apps/studio/data/query-client.ts`
- Studio explicitly retries 429 errors even on skip-retry pathnames, because failing immediately on rate limits causes refetch storms that amplify the problem → `apps/studio/data/query-client.ts`

## Where It Appears

| Aspect | PostgREST-js | Auth-js | Realtime-js | Studio (React Query) |
|--------|-------------|---------|-------------|---------------------|
| **Max retries** | 3 | 10 (token refresh) | Unlimited (reconnect) | 3 |
| **Backoff** | Exponential (1s, 2s, 4s, max 30s) | Fixed 200ms intervals | Stepped (1s, 2s, 5s, 10s) | Exponential (1s, 2s, 4s, max 30s) |
| **Retryable status codes** | 520, 503 | 502-504, 520-524, 530 | N/A (WebSocket) | All 5xx, 429 |
| **Non-retryable** | All non-GET/HEAD/OPTIONS, 4xx | 4xx (session removed) | N/A | 4xx except 429 |
| **Protocol** | HTTP | HTTP | WebSocket | HTTP |
| **Abort support** | AbortSignal-aware sleep | N/A | N/A | React Query cancellation |
| **Rate limit handling** | Not special-cased | Not special-cased | N/A | Retry-After header respected |
| **Session impact** | None | Non-retryable error removes session | Channel terminated | None |

## Design Intent

The retry patterns are calibrated to each client's use case. PostgREST-js is conservative: only idempotent reads retry, preventing accidental double-writes. Auth-js is aggressive on transient errors but terminal on auth-specific failures, because retrying an invalid refresh token is futile. Realtime-js has unlimited reconnect because WebSocket connections are expected to drop. Studio respects server backpressure (Retry-After) and avoids retry storms on expensive endpoints.

## Trade-offs

- **PostgREST-js method restriction**: Only retrying GET/HEAD/OPTIONS prevents accidental duplicate mutations but means POST/PATCH/DELETE failures are never auto-retried, even if they would be safe (e.g., idempotent upserts).
- **Auth-js session termination**: Aggressively removing the session on non-retryable errors ensures security but can cause UX disruption if the error is transient but mis-classified.
- **Realtime-js unbounded reconnect**: Unlimited reconnect with stepped intervals ensures persistent connections but could waste resources if the server is permanently unreachable.
- **Studio's rate-limit-aware retry**: Respecting Retry-After prevents request storms but means the UI appears unresponsive during rate limiting.

## Agent Guidance

- When debugging intermittent PostgREST failures, check whether the status code is 520 or 503. Only these trigger automatic retries. A 500 error will not be retried.
- Auth-js's `AuthRetryableFetchError` vs `AuthApiError` distinction is critical: retryable errors are transient (network), API errors are permanent (wrong credentials). If a user's session is unexpectedly removed, check for non-retryable auth errors.
- Realtime reconnect intervals are hardcoded at `[1000, 2000, 5000, 10000]` ms. Custom reconnect logic can be provided via the `reconnectAfterMs` option.
- In Studio, if a query to `/run-lints` or `/analytics/endpoints/logs.all` fails, it will NOT retry (unless it's a 429). This is intentional to avoid load amplification.

## Related

- [[PROC-CLIENT-SDK-AUTH]] -- Auth-js retry in token refresh
- [[PROC-STUDIO-AUTH]] -- Studio retry configuration
- [[PAT-CROSS-SERVICE-MIDDLEWARE]] -- Middleware that may trigger retryable errors
- [[PAT-CROSS-SERVICE-DATA-FLOW]] -- Data flow affected by retry behavior

