---
id: "PROC-AUTH-AUTH"
type: "process"
title: "Auth Authentication Mechanism"
domain: "supabase-reef"
status: draft
last_verified: 2026-04-15
freshness_note: "Deep scan of auth.go, tokens/service.go, hooks system, external OAuth flow, and MFA challenge pipeline"
freshness_triggers:
  - "internal/api/auth.go"
  - "internal/api/external.go"
  - "internal/api/hooks.go"
  - "internal/api/mfa.go"
  - "internal/api/token.go"
  - "internal/hooks/v0hooks/"
  - "internal/tokens/service.go"
known_unknowns:
  - "Exact session timebox and inactivity timeout default values (configured externally)"
  - "Full list of reserved JWT claims that the CustomAccessToken hook must not override"
  - "How SAML ACS flow integrates with session AAL tracking"
tags:
  - auth
  - jwt
  - mfa
  - oauth
  - hooks
aliases: []
relates_to:
  - type: "depends_on"
    target: "[[API-AUTH]]"
  - type: "parent"
    target: "[[SYS-AUTH]]"
  - type: "refines"
    target: "[[PROC-AUTH-ERROR-HANDLING]]"
sources:
  - category: "implementation"
    type: "github"
    ref: "internal/api/auth.go"
  - category: "implementation"
    type: "github"
    ref: "internal/api/external.go"
  - category: "implementation"
    type: "github"
    ref: "internal/api/hooks.go"
  - category: "implementation"
    type: "github"
    ref: "internal/api/mfa.go"
  - category: "implementation"
    type: "github"
    ref: "internal/api/token.go"
  - category: "implementation"
    type: "github"
    ref: "internal/hooks/v0hooks/manager.go"
  - category: "implementation"
    type: "github"
    ref: "internal/hooks/v0hooks/v0hooks.go"
  - category: "implementation"
    type: "github"
    ref: "internal/tokens/service.go"
notes: ""
---

## Purpose

Documents how Auth authenticates requests, issues JWTs, handles OAuth flows, enforces MFA, and supports extensibility via hooks. Auth is both the identity provider and a consumer of its own tokens for admin and user endpoints.

## Key Facts

- Bearer token extracted from Authorization header via regex `(?i)^bearer (\S+$)` in `bearerRegexp` → `internal/api/api.go`
- JWT parsing uses `jwt.NewParser` with `WithValidMethods` from config; supports key lookup by `kid` header with HS256 fallback for backward compatibility → `internal/api/auth.go`
- `requireAuthentication` middleware extracts token, parses claims, then loads user and session from database via `maybeLoadUserOrSession` → `internal/api/auth.go`
- Admin authorization checks JWT role against `config.JWT.AdminRoles` list; any matching role grants admin access → `internal/api/auth.go`
- Anonymous users are blocked from privileged operations via `requireNotAnonymous` which checks `claims.IsAnonymous` → `internal/api/auth.go`
- Access token claims include: sub, aud, email, phone, role, app_metadata, user_metadata, aal, amr, session_id, is_anonymous, client_id, scope → `internal/tokens/service.go`
- `CustomizeAccessToken` hook can modify JWT claims at issuance; output is validated against `MinimumViableTokenSchema` JSON Schema before signing → `internal/tokens/service.go`
- Seven hook types are supported: send-sms, send-email, customize-access-token, mfa-verification, password-verification, before-user-created, after-user-created → `internal/hooks/v0hooks/v0hooks.go`
- Hooks dispatch via two transports: HTTPS endpoints (`hookshttp`) or Postgres functions (`hookspgfunc`), selected by URI prefix → `internal/hooks/v0hooks/manager.go`
- OAuth flow creates a `FlowState` record in DB and uses its UUID as the `state` parameter; supports both PKCE and implicit flows → `internal/api/external.go`
- OAuth callback handles four account-linking decisions: LinkAccount, CreateAccount, AccountExists, MultipleAccounts (which errors) → `internal/api/external.go`
- Refresh token grant uses a 5-second retry loop with 10-30ms random backoff to avoid connection pool exhaustion on concurrent refreshes → `internal/tokens/service.go`
- Refresh token algorithm v2 uses HMAC-signed counter-based tokens per session, with configurable upgrade percentage from v1 → `internal/tokens/service.go`
- Session validity is checked against three conditions: timebox expiration, inactivity timeout, and low AAL (MFA not completed) → `internal/tokens/service.go`
- Token endpoint dispatches five grant types: password, refresh_token, id_token, pkce, and web3 → `internal/api/token.go`
- JWKS endpoint at `/.well-known/jwks.json` exposes only asymmetric public keys, filtering out HMAC keys → `internal/api/jwks.go`
- ID tokens follow OIDC spec with scope-filtered claims (openid, email, profile, phone) and 1-hour expiry; HS256 signing is explicitly rejected → `internal/tokens/service.go`
- Hooks are blocked from running inside database transactions; `checkTX` returns an error if `conn.TX != nil` → `internal/api/hooks.go`

## Steps

1. **Request arrives** -- middleware chain: recover, request ID, XFF, logging, body limit, optional timeout, optional tracing → `internal/api/api.go`
2. **Route matching** -- chi-style router selects handler; some routes add CAPTCHA or admin credential middleware
3. **Authentication check** -- `requireAuthentication` extracts Bearer token, parses JWT with key lookup (kid-based or HS256 fallback), validates signature
4. **Claims loading** -- parsed claims stored in context; user loaded from `sub` claim UUID, session loaded from `session_id` claim
5. **Admin check** -- for `/admin/*` endpoints, `requireAdmin` verifies JWT role is in `config.JWT.AdminRoles` list
6. **Handler executes** -- creates/modifies auth state (user, session, tokens, factors)
7. **Token issuance** -- `GenerateAccessToken` builds claims struct, invokes `CustomizeAccessToken` hook if enabled, validates output schema, signs JWT
8. **Refresh token** -- for v1: swap-and-revoke model with reuse interval; for v2: counter-based HMAC tokens with reuse detection heuristics (fail-to-save, concurrent-refresh)
9. **OAuth flow** -- redirect creates FlowState, callback loads state, resolves account linking decision, issues tokens or PKCE auth code
10. **MFA challenge** -- factor enrolled (TOTP, phone, WebAuthn), challenge created with expiry, verification invokes `mfa-verification` hook, session AAL upgraded on success

## Worked Examples

### OAuth PKCE Flow

1. Client calls `/authorize?provider=github&code_challenge=X&code_challenge_method=S256`
2. `GetExternalProviderRedirectURL` creates `FlowState` with PKCE params, persists to DB
3. User authenticates with GitHub, callback hits `/callback`
4. `loadExternalState` loads FlowState from UUID state param, checks expiration
5. `internalExternalProviderCallback` resolves account linking, updates FlowState with user ID and auth code
6. Client exchanges auth code via `grant_type=pkce` with code_verifier
7. Server verifies code_verifier against stored code_challenge, issues access + refresh tokens

### Refresh Token Reuse Detection (v2)

1. Client sends refresh request with counter-based token (counter=5)
2. Server loads session, finds current counter=6 (difference=1)
3. Heuristic: `likelyNotSavedByClient=true` (difference==1), reuse allowed
4. Server returns current active refresh token (counter=6) without incrementing
5. If difference > 1 and no concurrent refresh detected, session is terminated (rotation enabled) or request rejected

## Related

- [[SYS-AUTH]] -- parent system
- [[PROC-AUTH-ERROR-HANDLING]] -- error handling patterns used throughout auth
- [[API-AUTH]] -- API endpoints using this auth mechanism
