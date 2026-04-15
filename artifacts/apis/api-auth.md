---
id: "API-AUTH"
type: "api"
title: "Auth REST API"
domain: "auth"
status: "active"
last_verified: 2026-04-15
freshness_note: "Updated from OpenAPI spec and Go router source; covers all 59 operations across 43 paths"
freshness_triggers:
  - "internal/api/api.go"
  - "internal/api/apierrors/errorcode.go"
  - "openapi.yaml"
  - "sources/apis/auth/openapi.json"
known_unknowns:
  - "Exact rate limiting configuration per endpoint (configured via env vars, not in spec)"
  - "Webhook/hook extension points that modify request flow"
tags:
  - auth
  - rest-api
  - go
  - openapi
aliases: []
relates_to:
  - type: "parent"
    target: "[[SYS-AUTH]]"
  - type: "depends_on"
    target: "[[SCH-AUTH]]"
sources:
  - category: "implementation"
    type: "github"
    ref: "internal/api/api.go"
  - category: "implementation"
    type: "github"
    ref: "internal/api/apierrors/errorcode.go"
  - category: "documentation"
    type: "doc"
    ref: "sources/apis/auth/openapi.json"
notes: ""
---

## Overview

Auth exposes a REST API (OpenAPI 3.0.3) for authentication operations, served at `https://{project}.supabase.co/auth/v1`. The API surface spans 43 paths with 59 operations (24 POST, 21 GET, 6 PUT, 8 DELETE). Routes are organized in a Go chi-style router with middleware layers for CORS, XFF forwarding, request body limiting (1 MB), rate limiting, CAPTCHA verification, and request tracing. The API divides into public auth flows (signup, login, OAuth callbacks), token management, user self-service, admin endpoints requiring service_role credentials, an OAuth 2.1 server mode, and passkey/WebAuthn endpoints.

## Key Facts

- 43 paths / 59 operations; method split is POST 24, GET 21, PUT 6, DELETE 8 → `sources/apis/auth/openapi.json`
- Three security schemes: `APIKeyAuth` (apikey header), `UserAuth` (bearer JWT), `AdminAuth` (bearer service_role JWT) → `sources/apis/auth/openapi.json`
- `/token` supports 5 grant types: `password`, `refresh_token`, `id_token`, `pkce`, `web3` (Solana + Ethereum) → `sources/apis/auth/openapi.json`
- CAPTCHA middleware (`verifyCaptcha`) applied to signup, token, recover, resend, magiclink, otp, and SSO endpoints → `internal/api/api.go`
- Request body globally limited to 1 MB (`limitRequestBody(1 << 20)`) → `internal/api/api.go`
- Anonymous signup dispatched via separate `SignupAnonymously` handler when email+phone are both empty → `internal/api/api.go`
- Logout supports 3 scopes via query param: `global` (all sessions), `local` (current), `others` (all except current) → `sources/apis/auth/openapi.json`
- Pagination on admin list endpoints (`/admin/users`, `/admin/audit`) uses `page` + `per_page` query params → `sources/apis/auth/openapi.json`
- Error responses use two shapes: `HTTPError` with `{code, error_code, msg}` for auth errors and `OAuthError` with `{error, error_description}` for OAuth flows → `internal/api/apierrors/apierrors.go`
- 80+ distinct error codes defined in `errorcode.go`, from `email_exists` to `web3_unsupported_chain` → `internal/api/apierrors/errorcode.go`
- OAuth server endpoints (under `/oauth/`) gated by `requireOAuthServerEnabled` middleware; experimental → `internal/api/api.go`
- Passkey endpoints (under `/passkeys/`) gated by `requirePasskeyEnabled` middleware with registration, authentication, list, update, and delete operations → `internal/api/api.go`

## Source of Truth

OpenAPI spec at `sources/apis/auth/openapi.json` (canonical). Route definitions in `internal/api/api.go` (lines 164-420+). Each handler is a separate file in `internal/api/`.

## Resource Map

| Group | Method | Path | Handler | Auth |
|-------|--------|------|---------|------|
| Health | GET | `/health` | HealthCheck | None |
| Discovery | GET | `/.well-known/jwks.json` | WellKnownJwks | None |
| Discovery | GET | `/.well-known/openid-configuration` | WellKnownOpenID | None |
| Discovery | GET | `/.well-known/oauth-authorization-server` | WellKnownOpenID | OAuthServer |
| Auth | POST | `/signup` | Signup / SignupAnonymously | CAPTCHA |
| Auth | POST | `/token?grant_type={password,refresh_token,id_token,pkce,web3}` | Token | CAPTCHA |
| Auth | GET | `/authorize` | ExternalProviderRedirect | APIKey |
| Auth | GET/POST | `/callback` | ExternalProviderCallback | FlowState |
| Auth | POST | `/recover` | Recover | CAPTCHA |
| Auth | POST | `/magiclink` | MagicLink | CAPTCHA |
| Auth | POST | `/otp` | Otp | CAPTCHA |
| Auth | GET/POST | `/verify` | Verify | APIKey |
| Auth | POST | `/resend` | Resend | CAPTCHA |
| Auth | POST | `/reauthenticate` | Reauthenticate | Bearer JWT |
| Auth | POST | `/logout?scope={global,local,others}` | Logout | Bearer JWT |
| User | GET | `/user` | UserGet | Bearer JWT |
| User | PUT | `/user` | UserUpdate | Bearer JWT |
| User | GET | `/user/identities/authorize` | LinkIdentity | Bearer JWT |
| User | DELETE | `/user/identities/{identity_id}` | DeleteIdentity | Bearer JWT |
| User | GET | `/user/oauth/grants` | UserListOAuthGrants | Bearer JWT + OAuthServer |
| User | DELETE | `/user/oauth/grants` | UserRevokeOAuthGrant | Bearer JWT + OAuthServer |
| MFA | POST | `/factors` | EnrollFactor | Bearer JWT |
| MFA | POST | `/factors/{factor_id}/challenge` | ChallengeFactor | Bearer JWT |
| MFA | POST | `/factors/{factor_id}/verify` | VerifyFactor | Bearer JWT |
| MFA | DELETE | `/factors/{factor_id}` | UnenrollFactor | Bearer JWT |
| Passkey | POST | `/passkeys/authentication/options` | PasskeyAuthenticationOptions | CAPTCHA |
| Passkey | POST | `/passkeys/authentication/verify` | PasskeyAuthenticationVerify | None |
| Passkey | POST | `/passkeys/registration/options` | PasskeyRegistrationOptions | Bearer JWT |
| Passkey | POST | `/passkeys/registration/verify` | PasskeyRegistrationVerify | Bearer JWT |
| Passkey | GET | `/passkeys` | PasskeyList | Bearer JWT |
| Passkey | PATCH | `/passkeys/{passkey_id}` | PasskeyUpdate | Bearer JWT |
| Passkey | DELETE | `/passkeys/{passkey_id}` | PasskeyDelete | Bearer JWT |
| SSO | POST | `/sso` | SingleSignOn | CAPTCHA |
| SSO | GET | `/sso/saml/metadata` | SAMLMetadata | None |
| SSO | POST | `/sso/saml/acs` | SamlAcs | SAML |
| Admin | POST | `/invite` | Invite | service_role |
| Admin | GET | `/admin/audit` | adminAuditLog | service_role |
| Admin | GET | `/admin/users` | adminUsers | service_role |
| Admin | POST | `/admin/users` | adminUserCreate | service_role |
| Admin | GET/PUT/DELETE | `/admin/users/{user_id}` | adminUser* | service_role |
| Admin | GET/DELETE/PUT | `/admin/users/{user_id}/factors/{factor_id}` | adminUserFactor* | service_role |
| Admin | POST | `/admin/generate_link` | adminGenerateLink | service_role |
| Admin | CRUD | `/admin/sso/providers` | SSO provider management | service_role |
| Admin | CRUD | `/admin/oauth/clients` | OAuth client management | service_role |
| Admin | POST | `/admin/oauth/clients/{client_id}/regenerate_secret` | Regenerate secret | service_role |
| Admin | CRUD | `/admin/custom-providers` | Custom OAuth/OIDC provider management | service_role |
| OAuth Server | POST | `/oauth/clients/register` | Dynamic client registration | None |
| OAuth Server | POST | `/oauth/token` | OAuth 2.1 token | Client auth |
| OAuth Server | GET | `/oauth/authorize` | OAuth 2.1 authorize | Bearer JWT |
| OAuth Server | GET | `/oauth/authorizations/{id}` | Get authorization | Bearer JWT |
| OAuth Server | POST | `/oauth/authorizations/{id}/consent` | Consent | Bearer JWT |
| Settings | GET | `/settings` | Settings | None |

## Worked Example: Password Login via /token

**Request:**
```http
POST /auth/v1/token?grant_type=password HTTP/1.1
Host: abcdefghijklmnopqrst.supabase.co
apikey: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "password1"
}
```

**Response (200 OK):**
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIs...",
  "token_type": "bearer",
  "expires_in": 3600,
  "expires_at": 1713200000,
  "refresh_token": "4nYUCw0wZR_DNOTSDbSGMQ",
  "user": {
    "id": "d0714948-a2f3-4a32-b880-4e67e0e2a704",
    "email": "user@example.com",
    "role": "authenticated",
    "email_confirmed_at": "2026-04-10T12:00:00Z",
    "last_sign_in_at": "2026-04-15T08:30:00Z",
    "app_metadata": { "provider": "email", "providers": ["email"] },
    "user_metadata": {},
    "identities": [
      {
        "identity_id": "d0714948-a2f3-4a32-b880-4e67e0e2a704",
        "id": "d0714948-a2f3-4a32-b880-4e67e0e2a704",
        "provider": "email",
        "email": "user@example.com",
        "created_at": "2026-04-10T12:00:00Z",
        "updated_at": "2026-04-15T08:30:00Z"
      }
    ],
    "created_at": "2026-04-10T12:00:00Z",
    "updated_at": "2026-04-15T08:30:00Z"
  }
}
```

**Error (400 Bad Request):**
```json
{
  "code": 400,
  "error_code": "invalid_credentials",
  "msg": "Invalid login credentials"
}
```

## Related

- [[SYS-AUTH]] -- parent system
- [[SCH-AUTH]] -- data model backing these endpoints
