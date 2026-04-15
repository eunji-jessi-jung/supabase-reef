---
id: "SYS-AUTH"
type: "system"
title: "Supabase Auth Service"
domain: "auth"
status: "draft"
last_verified: 2026-04-15
freshness_note: "Updated with dependency, runtime, and behavioral detail from source inspection on 2026-04-15"
freshness_triggers:
  - "cmd/serve_cmd.go"
  - "docker-compose-dev.yml"
  - "go.mod"
  - "internal/api/api.go"
  - "internal/api/apiworker/apiworker.go"
  - "internal/conf/configuration.go"
  - "internal/models/user.go"
  - "internal/observability/tracing.go"
  - "internal/reloader/reloader.go"
  - "internal/tokens/service.go"
  - "migrations/20260302000000_add_passkeys.up.sql"
known_unknowns:
  - "Retry and error recovery strategy for external provider OAuth callbacks"
  - "Rate limiting thresholds and configuration details beyond code structure"
  - "Deployment topology in production (sidecar, standalone, multi-region)"
  - "How the hooks system interacts with Supabase platform-level orchestration"
  - "Full scope of the OAuth server feature (recently added, still evolving)"
  - "IndexWorker role and scheduling details beyond configuration presence"
tags:
  - auth
  - go
  - jwt
  - mfa
  - oauth
  - passkeys
  - saml
aliases:
  - "GoTrue"
  - "gotrue"
relates_to:
  - type: "refines"
    target: "[[API-AUTH]]"
  - type: "refines"
    target: "[[CON-AUTH-CLIENT-SDK]]"
  - type: "refines"
    target: "[[CON-AUTH-REALTIME]]"
  - type: "refines"
    target: "[[CON-AUTH-STORAGE]]"
  - type: "refines"
    target: "[[GLOSSARY-AUTH]]"
  - type: "refines"
    target: "[[PROC-AUTH-AUTH]]"
  - type: "refines"
    target: "[[RISK-AUTH-KNOWN-GAPS]]"
  - type: "refines"
    target: "[[SCH-AUTH]]"
  - type: "feeds"
    target: "[[SYS-CLIENT-SDK]]"
  - type: "feeds"
    target: "[[SYS-REALTIME]]"
  - type: "feeds"
    target: "[[SYS-STORAGE]]"
sources:
  - category: "implementation"
    type: "github"
    ref: "cmd/serve_cmd.go"
  - category: "implementation"
    type: "github"
    ref: "docker-compose-dev.yml"
  - category: "implementation"
    type: "github"
    ref: "go.mod"
  - category: "implementation"
    type: "github"
    ref: "internal/api/api.go"
  - category: "implementation"
    type: "github"
    ref: "internal/api/apiworker/apiworker.go"
  - category: "implementation"
    type: "github"
    ref: "internal/conf/configuration.go"
  - category: "implementation"
    type: "github"
    ref: "internal/models/user.go"
  - category: "implementation"
    type: "github"
    ref: "internal/observability/tracing.go"
  - category: "implementation"
    type: "github"
    ref: "internal/tokens/service.go"
  - category: "implementation"
    type: "github"
    ref: "main.go"
notes: ""
---

## Overview

Auth is the authentication and user management server for the Supabase platform. Written in Go (module `github.com/supabase/auth`), it is originally forked from Netlify's GoTrue but has diverged significantly. It issues JWTs consumed by all other Supabase services (Storage, Realtime, PostgREST) to enforce Row Level Security. The server handles the full spectrum of authentication methods: email/password, magic links, phone/SMS OTP, anonymous sign-in, external OAuth providers (Google, Apple, GitHub, Discord, and many more), SAML SSO, custom OAuth/OIDC providers, and passkeys/WebAuthn. It also provides MFA via TOTP, phone, and WebAuthn. Auth can optionally act as an OAuth 2.1 authorization server itself.

## Key Facts

- Go HTTP server using chi-based routing (`go-chi/chi/v5`) with a layered middleware chain: recoverer, request ID, Sb-Forwarded-For parsing, structured logging, XFF, body limit (1 MB), optional request timeout, optional request tracing, optional DB cleanup, and optional background email sending → `internal/api/api.go`
- Entry point embeds all migration SQL files via `//go:embed migrations/*` and delegates to a cobra CLI (`cmd.RootCommand().ExecuteContext`) with graceful shutdown on SIGTERM/SIGHUP/SIGINT → `main.go`
- Serve command opens a Postgres connection via `storage.DialContext`, creates the API with functional options (limiter, mailer), binds an `http.Server` with `ReadHeaderTimeout: 2s` to mitigate Slowloris, and uses `SO_REUSEPORT` for the TCP listener → `cmd/serve_cmd.go`
- Background `apiworker.Worker` goroutine runs alongside the HTTP server, receives config reload notifications, and coordinates async tasks (e.g. background email sending, index worker) → `internal/api/apiworker/apiworker.go`
- Hot-reload system: a `reloader.AtomicHandler` wraps the API so a new `API` instance can be swapped in at runtime; a `reloader.Reloader` watches a config directory via fsnotify or polling, with configurable grace period to debounce bursts → `cmd/serve_cmd.go`, `internal/reloader/`
- 40+ data models across `internal/models/` including User, Identity, Session, Factor, RefreshToken, OAuthClient, OAuthAuthorization, OAuthConsent, FlowState, WebAuthnCredential, WebAuthnChallenge, CustomOAuthProvider, and Web3 → `internal/models/`
- Token service issues JWTs with `AccessTokenClaims` containing email, phone, role, AAL, AMR entries, and session ID; supports configurable signing keys (RS256, HS256, ES256) and publishes a JWKS endpoint at `/.well-known/jwks.json` → `internal/tokens/service.go`, `internal/api/api.go`
- Hooks system uses three driver implementations: `v0hooks.Manager` orchestrates, `hookshttp` calls external HTTP endpoints, `hookspgfunc` invokes Postgres functions — allowing customization of auth behavior at defined extension points → `internal/hooks/`
- 25+ external OAuth provider implementations in `internal/api/provider/` including Google, Apple, GitHub, Discord, Azure, Slack, LinkedIn, X, plus a generic `custom_oauth.go` for admin-registered providers → `internal/api/provider/`
- 69 database migrations embedded in the binary, spanning 2021 to 2026, with the most recent adding passkeys (2026-03-02), custom OAuth providers (2026-02-19), and OAuth client states (2025-12-01) → `migrations/`
- Configuration loaded entirely from environment variables via `envconfig` with a `GlobalConfiguration` struct covering API, DB, JWT, MFA, SAML, WebAuthn, Passkey, OAuth server, SMTP, rate limits, sessions, CORS, tracing, metrics, profiling, and experimental flags → `internal/conf/configuration.go`
- HIBP (Have I Been Pwned) integration with optional bloom filter cache for password breach checking; configurable fail-open/fail-closed behavior → `internal/conf/configuration.go`, `internal/api/api.go`
- Observability stack: structured JSON logging (logrus), OpenTelemetry tracing/metrics export via OTLP (gRPC and HTTP), Prometheus metrics endpoint, Go runtime instrumentation, and optional profiler → `internal/observability/`, `go.mod`
- CAPTCHA verification layer supports pluggable verifiers configured via `SecurityConfiguration` → `internal/security/captcha.go`, `internal/api/api.go`
- Docker image built as a multi-stage Alpine build; final image runs as non-root user `supabase` (UID 1000); binary symlinked as both `auth` and `gotrue` for backward compatibility → `Dockerfile`

## Responsibilities

- **Owns**: User identity, authentication sessions, JWT issuance, MFA factors, OAuth provider integrations, SAML/SSO, refresh token rotation, password management, email/SMS verification flows, passkey/WebAuthn registration and authentication, optional OAuth 2.1 authorization server, custom OAuth/OIDC provider registry, audit logging of auth events
- **Does not own**: Authorization policy definitions (those are RLS policies in user databases), API gateway routing, object storage, real-time messaging, PostgREST query execution, client-side SDK logic (auth-js wraps the API but lives in a separate repo)

## Does NOT Own

| Boundary | Owner | How Auth Relates |
|---|---|---|
| Row Level Security policies | User-defined in Postgres | Auth issues the JWT that Postgres evaluates against RLS policies |
| API gateway / routing | Kong or Supabase Edge | Auth is one upstream service behind the gateway |
| Object storage | Storage service | Storage validates Auth-issued JWTs for access control |
| Real-time channels | Realtime service | Realtime validates Auth-issued JWTs for WebSocket auth |
| Client SDK logic | supabase-js / auth-js | SDK wraps Auth HTTP API; Auth has no knowledge of SDK internals |
| Database schema beyond `auth` namespace | Application developer | Auth only manages tables in the `auth` schema (configurable via `DB_NAMESPACE`) |

## Dependencies

| System | Integration Type | Purpose | Auth Method |
|---|---|---|---|
| PostgreSQL | Direct SQL (pgx/jackc driver via pop/v6) | Persistent storage for users, sessions, tokens, factors, SSO providers, migrations | Connection string (`DATABASE_URL`) |
| External OAuth providers (Google, Apple, GitHub, etc.) | HTTP REST / OIDC Discovery | Federate sign-in to 25+ identity providers | OAuth2 client credentials per provider |
| SMTP server | SMTP (gomail.v2) | Send verification, recovery, magic-link, and invite emails | SMTP credentials in config |
| SMS providers | HTTP (provider-specific) | Send OTP codes for phone auth and phone MFA | API keys per SMS provider |
| HIBP API | HTTP REST | Check passwords against known breaches | User-Agent header (no auth key) |
| OpenTelemetry collector | OTLP gRPC / HTTP | Export traces and metrics | Endpoint URL (unauthenticated by default) |
| Prometheus | HTTP scrape | Expose `/metrics` endpoint | None (pull-based) |
| CAPTCHA services | HTTP | Verify captcha tokens on signup/recovery/OTP endpoints | Provider-specific secret keys |
| Standard Webhooks endpoints | HTTP | Fire custom auth hooks to external services | Webhook signing secret (symmetric or asymmetric) |

## Domain Behavior Highlights

- **Atomic handler swap for zero-downtime config reload**: The `reloader.AtomicHandler` wraps the active `API` instance in an `atomic.Value`. When the config watcher detects changes, a brand-new `API` is constructed and atomically stored, so in-flight requests complete on the old instance while new requests hit the updated one → `cmd/serve_cmd.go`
- **Embedded migrations via `//go:embed`**: All 69 migration SQL files are compiled into the binary at build time, eliminating the need to ship migration files alongside the binary or mount them at runtime (though the path is overridable via `GOTRUE_DB_MIGRATIONS_PATH`) → `main.go`, `Dockerfile`
- **Functional options pattern for API construction**: `NewAPIWithVersion` accepts variadic `Option` values (`WithLimiter`, `WithMailer`) enabling the serve command and tests to inject different mailers, rate limiters, or other dependencies without changing the constructor signature → `internal/api/api.go`
- **Graceful shutdown with two-phase context**: The main goroutine creates a signal-aware `execCtx` for the serve command, and a separate `shutdownCtx` with a 1-minute timeout for observability cleanup (trace/metric exporters). The serve command itself further splits into `baseCtx` (for DB availability during drain) and shutdown of `http.Server` → `main.go`, `cmd/serve_cmd.go`
- **Background worker co-process**: The `apiworker.Worker` runs as a sibling goroutine to the HTTP server, sharing the same DB connection and template cache. It handles async tasks and receives config-reload notifications via a channel, ensuring it stays in sync with the latest configuration → `cmd/serve_cmd.go`, `internal/api/apiworker/apiworker.go`

## Runtime Components

| Component | Tech Stack | Purpose | Entry Point |
|---|---|---|---|
| HTTP API server | Go `net/http` + chi router | Serves all auth endpoints (signup, login, token, verify, admin, OAuth, SAML, passkeys) | `cmd/serve_cmd.go` → `httpSrv.Serve()` |
| Background worker | Go goroutine (`errgroup`) | Async email sending, index worker, other deferred tasks | `cmd/serve_cmd.go` → `wrk.Work(ctx)` |
| Config reloader | Go goroutine (fsnotify + poller) | Watches config directory and hot-swaps API instance | `cmd/serve_cmd.go` → `rl.Watch(ctx, fn)` |
| Token service | Go library | JWT issuance, signing key management, JWKS serving | `internal/tokens/service.go` |
| Hooks manager | Go library (v0hooks + HTTP/pgfunc drivers) | Dispatches auth event hooks to HTTP endpoints or Postgres functions | `internal/hooks/v0hooks/` |
| OAuth server (optional) | Go library | OAuth 2.1 authorization server: client registration, authorize, consent, token | `internal/api/oauthserver/` |
| Observability stack | OpenTelemetry SDK, logrus, Prometheus client | Structured logging, distributed tracing, metrics export | `internal/observability/` |
| Database layer | pop/v6 + pgx/jackc driver | Connection pooling, migrations, model CRUD | `internal/storage/` |
| Mailer subsystem | gomail (SMTP), template cache | Renders and sends transactional emails with multiple client implementations | `internal/mailer/` |

## Core Concepts

- **User**: The primary identity entity with email, phone, metadata, and ban/delete state
- **Identity**: Links a User to an auth provider (email, phone, OAuth provider). A user can have multiple identities.
- **Session**: Represents an authenticated session with AAL (Authenticator Assurance Level) tracking for MFA
- **Factor**: An MFA factor (TOTP, phone, WebAuthn) linked to a user
- **FlowState**: Tracks in-progress auth flows (PKCE, implicit, magic link) across redirects
- **OAuthClient**: Registered OAuth clients when Auth acts as an OAuth2 authorization server
- **Hooks**: Extension points that fire during auth events, implemented as HTTP calls or Postgres function calls
- **Passkey**: WebAuthn credential registered for passwordless authentication, managed via dedicated CRUD endpoints

## Related

- [[SCH-AUTH]] -- data model details
- [[API-AUTH]] -- API endpoint reference
- [[PROC-AUTH-AUTH]] -- authentication mechanism
- [[GLOSSARY-AUTH]] -- domain terminology
- [[RISK-AUTH-KNOWN-GAPS]] -- known risks and gaps
- [[CON-AUTH-STORAGE]] -- contract: Auth ↔ Storage JWT integration
- [[CON-AUTH-REALTIME]] -- contract: Auth ↔ Realtime JWT integration
- [[CON-AUTH-CLIENT-SDK]] -- contract: Auth ↔ Client SDK integration
- [[SYS-STORAGE]] -- consumes Auth JWTs for RLS
- [[SYS-REALTIME]] -- consumes Auth JWTs for WebSocket auth
- [[SYS-CLIENT-SDK]] -- auth-js sub-package wraps Auth API
