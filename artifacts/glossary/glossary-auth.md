---
id: "GLOSSARY-AUTH"
type: "glossary"
title: "Auth Domain Glossary"
domain: "auth"
status: "active"
last_verified: 2026-04-15
freshness_note: "Deep pass: added 17 terms, code locations, and disambiguation rows from model files, hooks, conf, tokens, and existing artifacts"
freshness_triggers:
  - "internal/models/"
  - "internal/api/"
  - "internal/hooks/"
  - "internal/conf/configuration.go"
  - "internal/tokens/service.go"
  - "internal/reloader/"
  - "internal/security/"
known_unknowns:
  - "Some terms may have business meanings not captured in code"
  - "Web3 provider support scope (Solana/Ethereum confirmed, others unclear)"
tags:
  - glossary
aliases: []
relates_to:
  - type: "parent"
    target: "[[SYS-AUTH]]"
  - type: "refines"
    target: "[[GLOSSARY-SUPABASE-REEF]]"
  - type: "integrates_with"
    target: "[[SCH-AUTH]]"
  - type: "integrates_with"
    target: "[[API-AUTH]]"
sources:
  - category: "implementation"
    type: "code"
    ref: "internal/models/user.go"
  - category: "implementation"
    type: "code"
    ref: "internal/models/factor.go"
  - category: "implementation"
    type: "code"
    ref: "internal/models/sessions.go"
  - category: "implementation"
    type: "code"
    ref: "internal/models/identity.go"
  - category: "implementation"
    type: "code"
    ref: "internal/models/flow_state.go"
  - category: "implementation"
    type: "code"
    ref: "internal/models/linking.go"
  - category: "implementation"
    type: "code"
    ref: "internal/models/refresh_token.go"
  - category: "implementation"
    type: "code"
    ref: "internal/models/one_time_token.go"
  - category: "implementation"
    type: "code"
    ref: "internal/models/oauth_client.go"
  - category: "implementation"
    type: "code"
    ref: "internal/models/custom_oauth_provider.go"
  - category: "implementation"
    type: "code"
    ref: "internal/models/sso.go"
  - category: "implementation"
    type: "code"
    ref: "internal/models/webauthn_credential.go"
  - category: "implementation"
    type: "code"
    ref: "internal/hooks/v0hooks/v0hooks.go"
  - category: "implementation"
    type: "code"
    ref: "internal/hooks/v0hooks/manager.go"
  - category: "implementation"
    type: "code"
    ref: "internal/hooks/hookshttp/hookshttp.go"
  - category: "implementation"
    type: "code"
    ref: "internal/hooks/hookspgfunc/hookspgfunc.go"
  - category: "implementation"
    type: "code"
    ref: "internal/conf/configuration.go"
  - category: "implementation"
    type: "code"
    ref: "internal/tokens/service.go"
  - category: "implementation"
    type: "code"
    ref: "internal/reloader/handler.go"
  - category: "implementation"
    type: "code"
    ref: "internal/security/captcha.go"
notes: ""
---

## Terms

| Term | Definition | Code Location | See Also |
|------|-----------|---------------|----------|
| GoTrue | Original name of the auth server, forked from Netlify's GoTrue. Still appears in config env vars (GOTRUE_*), the Docker binary symlink (`gotrue`), and legacy references. | `cmd/root_cmd.go`, `Dockerfile` | [[SYS-AUTH]] |
| AAL | Authenticator Assurance Level. Tracks MFA step-up: aal1 = single factor, aal2 = multi-factor verified, aal3 = hardware key verified. Stored as an iota enum on Session. | `internal/models/sessions.go` | |
| Factor | An MFA credential (TOTP, phone, WebAuthn) enrolled by a user. Has status: `unverified` or `verified`. Three factor types defined as string constants. | `internal/models/factor.go` | |
| Identity | A link between a User and an authentication provider. Users can have multiple identities (e.g., email + Google). JSON marshaling swaps `id`/`identity_id` for backward compatibility with gotrue-js. | `internal/models/identity.go` | [[SCH-AUTH]] |
| FlowState | Tracks an in-progress authentication flow (PKCE, implicit, magic link) across HTTP redirects. Stores code_challenge, provider tokens, invite token, referrer, and OAuth client state. | `internal/models/flow_state.go` | |
| PKCE | Proof Key for Code Exchange. OAuth2 extension used for secure token exchange in public clients. FlowState stores code_challenge and code_challenge_method for PKCE flows. | `internal/api/token.go`, `internal/models/flow_state.go` | |
| service_role | A privileged JWT role that bypasses RLS. Used by admin endpoints and server-side operations. Distinct from the default `authenticated` role assigned to regular users. | `internal/api/auth.go` | [[GLOSSARY-SUPABASE-REEF]] |
| Hooks | Extension points fired during auth events. Dispatched via a dual-transport Manager to either HTTP webhooks or Postgres function calls. 7 named hook points defined in v0hooks. | `internal/hooks/v0hooks/`, `internal/hooks/hookshttp/`, `internal/hooks/hookspgfunc/` | [[DEC-AUTH-DUAL-TRANSPORT-HOOKS]] |
| OAuthClient | A registered client when Auth acts as an OAuth 2.1 authorization server. Has client_type (public/confidential) and token_endpoint_auth_method. Not to be confused with external OAuth providers. | `internal/models/oauth_client.go` | |
| AMR | Authentication Methods Reference. Records which methods were used in a session (password, otp, oauth, mfa, etc.). 16 method values defined. Stored as AMRClaim entries on sessions. | `internal/models/amr.go`, `internal/models/factor.go` | |
| WebAuthn | Web Authentication standard for passwordless auth via passkeys/FIDO2 security keys. Credentials stored separately from MFA factors in `webauthn_credentials` table. | `internal/models/webauthn_credential.go` | |
| Captcha | CAPTCHA verification (Turnstile, hCaptcha) applied to signup, token, recover, resend, magiclink, otp, and SSO endpoints. Pluggable verifier configured via SecurityConfiguration. | `internal/security/captcha.go`, `internal/api/api.go` | |
| HIBP | Have I Been Pwned. Optional password breach checking with bloom filter cache. Configurable fail-open/fail-closed behavior via HIBP settings in configuration. | `internal/conf/configuration.go`, `internal/api/api.go` | |
| Account Linking | The process of connecting a new identity provider to an existing user account. DetermineAccountLinking computes one of four decisions: AccountExists, CreateAccount, LinkAccount, or MultipleAccounts. | `internal/models/linking.go` | |
| Linking Domain | A runtime string grouping identities that should belong to the same user. SSO providers get their own domain; all other providers share the `default` domain. Not persisted in DB. | `internal/models/linking.go` | |
| RefreshToken | Opaque token enabling session extension. Uses BIGINT auto-increment PK (not UUID). Supports rotation via `parent` column chain tracking and `revoked` flag. | `internal/models/refresh_token.go` | [[SCH-AUTH]] |
| OneTimeToken | Short-lived token for verification flows. 6 types: confirmation, reauthentication, recovery, email_change_new, email_change_current, phone_change. Stored with hash, not plaintext. | `internal/models/one_time_token.go` | |
| SSOProvider | Container entity for SAML-based Single Sign-On providers. Has a 1:1 SAMLProvider child and 0..N SSODomain children for email domain mapping. Can be disabled via boolean flag. | `internal/models/sso.go` | |
| SAMLProvider | SAML identity provider configuration stored as a child of SSOProvider. Contains IdP metadata, entity ID, attribute mappings, and name ID format. | `internal/models/sso.go` | |
| CustomOAuthProvider | Admin-registered OAuth2 or OIDC provider (as opposed to the built-in 25+ providers). Has provider_type (`oauth2`/`oidc`), PKCE toggle, attribute mapping, and optional OIDC discovery caching. | `internal/models/custom_oauth_provider.go` | [[SCH-AUTH]] |
| OAuthServerConsent | Records a user's consent to grant scopes to an OAuthClient. Part of the OAuth 2.1 authorization server feature (experimental). | `internal/models/oauth_consent.go` | |
| OAuthServerAuthorization | Represents an active authorization grant in the OAuth 2.1 server. Links a user, client, and granted scopes with an authorization code. | `internal/models/oauth_authorization.go` | |
| OAuthClientState | Tracks state across OAuth authorization flows when Auth acts as an OAuth server. Prevents CSRF by binding state to a session. | `internal/models/oauth_client_state.go` | |
| ExtensibilityPoint | A single hook configuration entry with URI, enabled toggle, and HTTP signing secrets. URI scheme determines transport: `pg-functions:` for Postgres, `http:`/`https:` for webhooks. | `internal/conf/configuration.go` | [[DEC-AUTH-DUAL-TRANSPORT-HOOKS]] |
| AtomicHandler | A `sync/atomic.Value`-wrapped HTTP handler enabling zero-downtime config reload. The reloader constructs a new API instance and atomically swaps it in while in-flight requests complete on the old one. | `internal/reloader/handler.go` | [[SYS-AUTH]] |
| Standard Webhooks | The signing specification used for HTTP hooks. Sets `webhook-id`, `webhook-timestamp`, `webhook-signature` headers using symmetric HMAC signatures from the standard-webhooks library. | `internal/hooks/hookshttp/hookshttp.go` | [[DEC-AUTH-DUAL-TRANSPORT-HOOKS]] |
| Web3 | Blockchain wallet authentication (Solana, Ethereum). Uses `web3` as both provider name and grant type. Defined as an AuthenticationMethod. | `internal/models/web3.go`, `internal/models/factor.go` | |
| instance_id | Deprecated UUID column on users, refresh_tokens, and audit_log_entries. Always compared against `uuid.Nil`. Legacy from multi-tenant GoTrue era. | `internal/models/user.go`, `internal/models/refresh_token.go` | |
| Passkey | FIDO2/WebAuthn credential registered for passwordless primary authentication (distinct from WebAuthn used as an MFA factor). Managed via dedicated CRUD endpoints gated by `requirePasskeyEnabled` middleware. | `internal/api/api.go`, `migrations/20260302000000_add_passkeys.up.sql` | [[API-AUTH]] |

## Disambiguation

| Term | Auth Meaning | Other Service Meaning |
|------|-------------|----------------------|
| Token | JWT access token or opaque refresh token issued by Auth | Storage: S3 presigned URL token; Realtime: channel access token (same JWT, different consumer) |
| Session | Authenticated user session with AAL tracking, stored in `auth.sessions` table | Realtime: a WebSocket connection session; Client SDK: `AuthSession` object wrapping token pair |
| Provider | An identity source (Google, GitHub, email, SAML SSO, custom OAuth) | Storage: S3 storage backend provider; Client SDK: auth provider UI component |
| Hook | Extensibility point with Postgres or HTTP transport, fired during auth events | Storage: no equivalent; Realtime: Postgres change notification (different mechanism) |
| Client | An OAuth2 client application registered with Auth's OAuth server (`OAuthServerClient`) | Client SDK: the Supabase client library (`supabase-js`) |
| Scope | OAuth2 scope string (openid, email, profile) granted to an OAuthClient | Realtime: broadcast/presence scope; Client SDK: not used |

## Naming Conventions

- Model files use snake_case Go filenames matching entity names (user.go, identity.go, sessions.go)
- API handler files named by feature (signup.go, token.go, mfa.go, admin.go)
- Config env vars use GOTRUE_ prefix (legacy) or AUTH_ prefix
- Database tables live in the `auth` schema (configurable via `DB_NAMESPACE`)
- Hook names use kebab-case strings (e.g., `send-sms`, `customize-access-token`)
- OAuth server models prefixed with `OAuthServer` to distinguish from external OAuth provider types

## Related

- [[SYS-AUTH]] -- parent system
- [[SCH-AUTH]] -- data model details
- [[API-AUTH]] -- API endpoint reference
- [[DEC-AUTH-DUAL-TRANSPORT-HOOKS]] -- hooks architecture decision
- [[GLOSSARY-SUPABASE-REEF]] -- unified cross-service glossary
