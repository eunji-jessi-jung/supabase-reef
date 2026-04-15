---
id: "DEC-AUTH-DUAL-TRANSPORT-HOOKS"
type: "decision"
title: "Auth Hooks: Dual-Transport Extensibility via Postgres Functions and HTTP Webhooks"
domain: "auth"
status: "active"
last_verified: 2026-04-15
freshness_note: "Deep read of hooks package, manager dispatch, configuration, and all invocation call sites"
freshness_triggers:
  - "internal/hooks/v0hooks/manager.go"
  - "internal/hooks/v0hooks/v0hooks.go"
  - "internal/hooks/hookshttp/hookshttp.go"
  - "internal/hooks/hookspgfunc/hookspgfunc.go"
  - "internal/conf/configuration.go"
  - "internal/api/hooks.go"
known_unknowns:
  - "Whether async/fire-and-forget hooks are planned (current hooks are all synchronous)"
  - "Versioning strategy beyond v0hooks (package name implies future versions)"
  - "How hook latency impacts end-user-facing request latency in production at scale"
  - "Whether asymmetric webhook signing is fully implemented (TODO comment in hookshttp.go)"
tags:
  - decision
  - architecture
  - hooks
  - extensibility
aliases: []
relates_to:
  - type: "parent"
    target: "[[SYS-AUTH]]"
  - type: "integrates_with"
    target: "[[API-AUTH]]"
  - type: "integrates_with"
    target: "[[SCH-AUTH]]"
sources:
  - category: "implementation"
    type: "github"
    ref: "internal/hooks/v0hooks/manager.go"
  - category: "implementation"
    type: "github"
    ref: "internal/hooks/v0hooks/v0hooks.go"
  - category: "implementation"
    type: "github"
    ref: "internal/hooks/hookshttp/hookshttp.go"
  - category: "implementation"
    type: "github"
    ref: "internal/hooks/hookspgfunc/hookspgfunc.go"
  - category: "implementation"
    type: "github"
    ref: "internal/hooks/hookserrors/hookserrors.go"
  - category: "implementation"
    type: "github"
    ref: "internal/conf/configuration.go"
  - category: "implementation"
    type: "github"
    ref: "internal/api/hooks.go"
  - category: "implementation"
    type: "github"
    ref: "internal/api/api.go"
  - category: "context"
    type: "documentation"
    ref: "sources/context/decisions/architecture-overview.md"
    notes: "Supabase product principles: 'everything is extensible', 'everything works in isolation'"
notes: ""
---

## Context

Auth needs to be extensible so that platform operators and self-hosters can inject custom logic at well-defined points in the authentication lifecycle -- for example, enriching JWT claims, intercepting verification attempts, replacing the built-in email/SMS senders, or vetting new users before creation. The challenge is providing this extensibility without coupling Auth to a specific hosting environment, while keeping the latency and reliability characteristics acceptable for an authentication hot path. Supabase's architecture already has Postgres co-located with Auth, so leveraging Postgres functions is a natural zero-deployment option for many users. However, some use cases (external CRMs, analytics, third-party mailers) require calling out to HTTP services.

## Decision

Implement a **dual-transport hook system** with a single `Manager` facade that dispatches to either a **Postgres function** driver (`hookspgfunc`) or an **HTTP webhook** driver (`hookshttp`) based on the URI scheme configured for each extensibility point. Each hook is defined as a named extensibility point with typed input/output structs, a per-point enable toggle, and a URI that determines the transport. The Manager selects the driver at dispatch time by inspecting the URI prefix (`pg-functions:` vs `http:`/`https:`).

## Key Facts

- 7 named hook points defined as constants: `send-sms`, `send-email`, `customize-access-token`, `mfa-verification`, `password-verification`, `before-user-created`, `after-user-created` -> `internal/hooks/v0hooks/v0hooks.go`
- Manager facade holds both an `hookshttp.Dispatcher` and a `hookspgfunc.Dispatcher`; `dispatch()` selects driver by URI prefix at runtime -> `internal/hooks/v0hooks/manager.go`
- Postgres hook driver executes `SELECT schema.function_name(?)` inside an existing transaction (or opens a new one), with `SET LOCAL statement_timeout` to cap execution time (default 2 seconds) -> `internal/hooks/hookspgfunc/hookspgfunc.go`
- HTTP hook driver uses Standard Webhooks spec for signing: sets `webhook-id`, `webhook-timestamp`, `webhook-signature` headers using the `standard-webhooks` Go library -> `internal/hooks/hookshttp/hookshttp.go`
- HTTP hooks retry up to 3 times with 2-second backoff between attempts; 5-second per-attempt timeout; 200KB response payload limit -> `internal/hooks/hookshttp/hookshttp.go`
- Hook errors propagate via a shared `hookserrors.Error` type that carries `http_code` and `message`; error presence is determined by a non-empty `message` field in an `error` JSON envelope -> `internal/hooks/hookserrors/hookserrors.go`
- Configuration is entirely via environment variables: each hook point has `URI`, `Enabled`, and `HTTPHookSecrets` fields in `ExtensibilityPointConfiguration`; secrets use `|` separator for multiple keys -> `internal/conf/configuration.go`
- URI validation enforces that HTTP (non-TLS) hooks only work with localhost/127.0.0.1/::1/host.docker.internal; HTTPS hooks require valid signing secrets -> `internal/conf/configuration.go`
- Postgres hook URI format is `pg-functions://schema/function_name`; the schema and function name are validated against a regex and quoted into the SQL call -> `internal/conf/configuration.go`
- `before-user-created` and `after-user-created` hooks explicitly refuse to fire inside an open transaction (`checkTX`), requiring the caller to commit first -> `internal/api/hooks.go`
- The `customize-access-token` hook is invoked from the token service during JWT issuance, meaning it affects every token grant (password, refresh, OAuth, PKCE) -> `internal/tokens/service.go`, `internal/api/api.go`
- MFA and password verification hooks can return a `reject` decision that blocks the attempt and optionally logs the user out (`should_logout_user` for password hooks) -> `internal/hooks/v0hooks/v0hooks.go`
- The package is named `v0hooks`, signaling it is the initial hook version; no v1 or v2 packages exist yet -> `internal/hooks/v0hooks/`
- All hooks are synchronous and block the request until the hook returns or times out -- there is no async/fire-and-forget mode -> `internal/hooks/v0hooks/manager.go`

## Rationale

The dual-transport design serves two distinct deployment profiles. **Postgres functions** leverage the already-colocated database, eliminating network hops and infrastructure setup -- a Supabase user can write a `plpgsql` function and configure its URI without deploying any additional service. **HTTP webhooks** serve use cases requiring external system integration (third-party email providers, analytics pipelines, compliance systems) and follow the Standard Webhooks specification for interoperability. By keeping transport selection at the URI level and hiding it behind the Manager facade, the API layer (`internal/api/`) is insulated from transport details and invokes hooks identically regardless of backend.

This design embodies two of Supabase's product principles -> `sources/context/decisions/architecture-overview.md`:

- **Everything is extensible**: Rather than adding a bespoke plugin system, dual-transport hooks extend Auth behavior using existing primitives — standard Postgres functions and standard HTTP webhooks. Users customize auth lifecycle using tools they already know, not proprietary SDKs.
- **Everything works in isolation**: The Postgres hook transport requires nothing beyond the co-located database. A self-hosted user can extend Auth with `plpgsql` functions and zero additional infrastructure, satisfying the litmus test: "Can a user run this with just a Postgres database?"

## Consequences

### Positive
- Zero-deployment extensibility for Postgres-hosted users: write a function, flip a config toggle
- Standard Webhooks compliance for HTTP hooks means signing/verification tooling is reusable across ecosystems
- Typed input/output structs per hook name provide compile-time safety and clear API contracts for hook implementers
- Transaction-aware Postgres hooks can share the same DB transaction as the auth operation, enabling atomic side effects
- Centralized Manager with `Enabled()` check allows hooks to be no-ops with zero overhead when disabled

### Negative
- Synchronous execution means a slow hook directly increases auth endpoint latency; no circuit-breaker pattern is visible
- HTTP hook retry loop (up to 3 retries x 2s backoff) can add up to ~11 seconds to a request in worst case
- Postgres hooks share the Auth database connection pool; a slow hook function can exhaust connections and deadlock if conn is nil (the code warns about pool-exhaustion deadlocks in manager.go comments)
- Hook error semantics have quirks: an error response with `http_code: 500` but empty `message` is treated as success (noted as a BC concern in TODO comments)

### Neutral
- The `v0hooks` naming convention leaves room for a future hook API version without breaking the current one
- Hook secrets support multiple keys via `|` separator, enabling zero-downtime key rotation

## Related

- [[SYS-AUTH]] -- parent system overview
- [[API-AUTH]] -- API endpoints that invoke hooks
- [[SCH-AUTH]] -- data model (hooks can read/write via Postgres functions)
- [[GLOSSARY-AUTH]] -- domain terminology including Hooks, Factor, AMR
