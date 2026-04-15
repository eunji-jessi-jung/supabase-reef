---
id: "PAT-CROSS-SERVICE-MIDDLEWARE"
type: "pattern"
title: "Cross-Service Middleware Pattern"
domain: "supabase-reef"
status: draft
last_verified: 2026-04-15
freshness_note: "scuba-depth cross-service pattern analysis"
freshness_triggers:
  - "internal/api/middleware.go"
  - "internal/api/router.go"
  - "internal/api/api.go"
  - "src/http/plugins/jwt.ts"
  - "src/http/plugins/db.ts"
  - "src/http/plugins/tenant-id.ts"
  - "src/http/plugins/log-request.ts"
  - "apps/studio/data/fetchers.ts"
  - "packages/core/supabase-js/src/lib/fetch.ts"
  - "packages/core/auth-js/src/lib/fetch.ts"
known_unknowns:
  - "Full ordering of Auth's middleware chain beyond what's visible in api.go"
  - "Whether Storage plugins have explicit ordering dependencies beyond Fastify's registration order"
  - "Studio's Next.js middleware configuration for route protection"
tags:
  - pattern
  - cross-service
  - middleware
aliases:
  - "Request Pipeline Pattern"
relates_to:
  - type: "related"
    target: "[[PROC-AUTH-AUTH]]"
  - type: "related"
    target: "[[PROC-STORAGE-AUTH]]"
  - type: "related"
    target: "[[PROC-STUDIO-AUTH]]"
  - type: "related"
    target: "[[PROC-CLIENT-SDK-AUTH]]"
sources:
  - category: "implementation"
    type: "github"
    ref: "internal/api/middleware.go"
  - category: "implementation"
    type: "github"
    ref: "internal/api/router.go"
  - category: "implementation"
    type: "github"
    ref: "internal/api/api.go"
  - category: "implementation"
    type: "github"
    ref: "src/http/plugins/jwt.ts"
  - category: "implementation"
    type: "github"
    ref: "src/http/plugins/db.ts"
  - category: "implementation"
    type: "github"
    ref: "src/http/plugins/tenant-id.ts"
  - category: "implementation"
    type: "github"
    ref: "apps/studio/data/fetchers.ts"
  - category: "implementation"
    type: "github"
    ref: "packages/core/supabase-js/src/lib/fetch.ts"
  - category: "implementation"
    type: "github"
    ref: "packages/core/auth-js/src/lib/fetch.ts"
notes: ""
---

## Overview

All Supabase services implement middleware/plugin patterns for cross-cutting concerns (auth, logging, rate limiting, error handling), but use fundamentally different mechanisms based on their frameworks. Auth uses chi-style middleware with a custom `middlewareHandler` type that returns `(context.Context, error)`. Storage uses Fastify plugins with `preHandler`, `onSend`, `onTimeout`, and `onRequestAbort` hooks. Studio uses openapi-fetch middleware with `onRequest`/`onResponse` callbacks. The Client SDK uses a fetch wrapper pattern that decorates every outbound request.

## Key Facts

- Auth defines `middlewareHandler` as `func(w http.ResponseWriter, r *http.Request) (context.Context, error)` -- middleware either enriches context or returns an error that short-circuits via `HandleResponseError` → `internal/api/router.go`
- Auth's middleware chain includes: recover, request ID, XFF handling, logging, body limit (`limitRequestBody` using `http.MaxBytesReader`), optional timeout, optional tracing → `internal/api/api.go`
- Auth's `timeoutMiddleware` runs handlers in a goroutine with a `timeoutResponseWriter` that buffers the response; on deadline exceeded, it returns 504 Gateway Timeout → `internal/api/middleware.go`
- Auth rate limiting uses `tollbooth` library, keyed by the first value of a configurable rate limit header (e.g., X-Forwarded-For), with RFC 7230 comma-separated header handling → `internal/api/middleware.go`
- Auth supports feature-gate middleware: `requireSAMLEnabled`, `requireManualLinkingEnabled`, `requireOAuthServerEnabled`, `requireCustomOAuthEnabled`, `requirePasskeyEnabled` -- each returns 404 if the feature is disabled → `internal/api/middleware.go`
- Auth's `databaseCleanup` middleware runs after successful mutating requests (POST/PUT/PATCH/DELETE with 2xx response) to clean expired/stale rows → `internal/api/middleware.go`
- Storage uses named Fastify plugins (`auth-jwt`, `db-init`, `db-superuser-init`, `tenant-id`, `db-migrations`) that register `preHandler` hooks for request processing → `src/http/plugins/jwt.ts`, `src/http/plugins/db.ts`
- Storage's `tenant-id` plugin extracts tenant ID from `x-forwarded-host` header using regex in multi-tenant mode, defaulting to a configured tenant ID in single-tenant mode → `src/http/plugins/tenant-id.ts`
- Storage's DB plugin disposes connections on three hooks (`onSend`, `onTimeout`, `onRequestAbort`), ensuring cleanup even for failed or aborted requests → `src/http/plugins/db.ts`
- Studio's openapi-fetch client uses two middleware objects: one for request augmentation (`onRequest` adds auth headers and validates pg-meta connections) and one for response formatting (`onResponse` enriches error bodies with request ID and status) → `apps/studio/data/fetchers.ts`
- Studio's `pgMetaGuard` intercepts `/platform/pg-meta/` requests and throws 400 if `x-connection-encrypted` header is missing, preventing unnecessary backend hops → `apps/studio/data/fetchers.ts`
- Client SDK's `fetchWithAuth` is a fetch decorator that injects `apikey` and `Authorization: Bearer` headers on every request, acting as client-side middleware → `packages/core/supabase-js/src/lib/fetch.ts`
- Auth-js (GoTrueClient) adds `X-Supabase-Api-Version` and `X-Client-Info` headers to all auth-specific HTTP requests → `packages/core/auth-js/src/lib/fetch.ts`

## Where It Appears

| Aspect | Auth (Go/chi) | Storage (TS/Fastify) | Studio (React/openapi-fetch) | Client SDK (TS/fetch) |
|--------|--------------|---------------------|-----------------------------|-----------------------|
| **Framework** | chi middleware chain | Fastify plugin hooks | openapi-fetch middleware | fetch wrapper |
| **Signature** | `(w, r) → (ctx, error)` | `preHandler(request)` | `onRequest({request})` | `(input, init) → Response` |
| **Error handling** | Returns error, `HandleResponseError` serializes | Throws error, Fastify error handler catches | Throws `ResponseError` | Returns error in `{ data, error }` |
| **Auth injection** | `requireAuthentication` parses JWT to context | `auth-jwt` plugin sets `request.jwt`, `request.jwtPayload` | `constructHeaders()` adds Bearer token | `fetchWithAuth` adds apikey + Bearer |
| **Tenant resolution** | Single-tenant (from config) | `tenant-id` plugin from x-forwarded-host | Project ref in URL path | Supabase URL encodes project |
| **Rate limiting** | tollbooth per header value | N/A (handled upstream) | N/A | N/A |
| **Cleanup** | `databaseCleanup` after 2xx mutating requests | DB dispose on onSend/onTimeout/onRequestAbort | N/A | N/A |
| **Feature gating** | `require*Enabled` middleware per route | Fastify route config options | `IS_PLATFORM` flag | N/A |

## Design Intent

Each service's middleware pattern is idiomatic to its framework. Auth's chi middleware composes via `r.With(fn)` for per-route middleware and `r.Use(fn)` for global middleware. Storage's Fastify plugins use a hook lifecycle that guarantees cleanup (onSend, onTimeout, onRequestAbort). Studio's openapi-fetch middleware provides type-safe request/response transformation. The Client SDK's fetch wrapper is the simplest pattern, appropriate for a library that delegates actual request handling to the server.

## Trade-offs

- **Auth's goroutine timeout**: Running handlers in a goroutine with a deadline enables hard timeouts but adds complexity (buffered response writer, panic channel).
- **Storage's multi-hook cleanup**: Registering cleanup on three separate hooks (onSend, onTimeout, onRequestAbort) ensures no connection leaks but is verbose.
- **Studio's client-side middleware**: Limited to header manipulation and response formatting. Cannot enforce server-side concerns like rate limiting.
- **No shared middleware abstraction**: Each service implements middleware independently. Cross-cutting concerns like request ID generation are implemented separately in each service.

## Agent Guidance

- Auth middleware ordering matters: rate limiting must come before authentication to prevent unauthenticated requests from bypassing rate limits. Check the router setup in `api.go` for the exact ordering.
- In Storage, if a request hangs, check whether the DB connection was properly disposed. The three-hook pattern should handle it, but custom plugins that register their own resources need similar cleanup.
- Studio's `pgMetaGuard` is a circuit breaker for database queries without a valid connection string. If pg-meta requests fail with 400, check the `x-connection-encrypted` header.
- The Client SDK's `fetchWithAuth` always sets the `apikey` header even when a Bearer token is present. Both headers are required by the Supabase API gateway.

## Related

- [[PROC-AUTH-AUTH]] -- Auth middleware in authentication flow
- [[PROC-STORAGE-AUTH]] -- Storage plugin chain
- [[PROC-STUDIO-AUTH]] -- Studio middleware for API calls
- [[PROC-CLIENT-SDK-AUTH]] -- SDK fetch wrapper
