---
id: "PROC-AUTH-CUSTOM-OAUTH-PROVIDERS-LIFECYCLE"
type: process
title: "Auth Custom OAuth/OIDC Providers Entity Lifecycle"
domain: supabase-reef
status: draft
last_verified: 2026-04-15
freshness_note: "Derived from GoTrue internal/models/custom_oauth_provider.go and internal/api/custom_oauth_admin.go; current with source as of verification date."
freshness_triggers:
  - "internal/api/custom_oauth_admin.go"
  - "internal/models/custom_oauth_provider.go"
known_unknowns:
  - "No audit logging is implemented for provider CRUD operations (documented TODO in source)"
  - "Runtime authentication flow using custom providers partially traced: Provider() dispatches to loadCustomProvider(), callback uses OAuthProvider() for token exchange"
  - "OIDC discovery cache invalidation strategy during concurrent requests is unclear"
tags:
  - auth
  - entity-lifecycle
  - oauth
  - oidc
  - custom-providers
aliases:
  - "custom oauth providers lifecycle"
  - "auth.custom_oauth_providers lifecycle"
relates_to:
  - type: parent
    target: "[[SCH-AUTH]]"
  - type: refines
    target: "[[SYS-AUTH]]"
sources:
  - category: implementation
    type: github
    ref: "internal/api/custom_oauth_admin.go"
    notes: "Admin CRUD for custom OAuth/OIDC providers, validation, OIDC discovery"
  - category: implementation
    type: github
    ref: "internal/models/custom_oauth_provider.go"
    notes: "CustomOAuthProvider model, encrypted secrets, OIDC discovery cache, CRUD"
  - category: documentation
    type: doc
    ref: "sources/schemas/auth/schema.md"
    notes: "Canonical schema reference for custom_oauth_providers table"
---

## Purpose

Documents the lifecycle of rows in the `auth.custom_oauth_providers` table. These represent admin-configured OAuth2 or OIDC identity providers that extend the built-in provider set. Each provider stores client credentials, endpoint URLs, attribute mappings, and (for OIDC) a cached discovery document. This is an admin-only configuration entity.

## Key Facts

- `ProviderType` is a string enum with two values: `"oauth2"` and `"oidc"` -> `internal/models/custom_oauth_provider.go`
- `identifier` must start with `"custom:"` prefix (e.g., `"custom:myidp"`) and is unique across all providers -> `internal/api/custom_oauth_admin.go`
- `client_secret` is encrypted via `crypto.NewEncryptedString` using the provider ID as AAD; falls back to plaintext when encryption is disabled -> `internal/models/custom_oauth_provider.go`
- OIDC providers require `issuer` field; OAuth2 providers require `authorization_url`, `token_url`, and `userinfo_url` -> `internal/api/custom_oauth_admin.go`
- On OIDC provider creation, the discovery document is fetched and validated before persisting; required fields are `issuer`, `authorization_endpoint`, `token_endpoint`, `jwks_uri` -> `internal/api/custom_oauth_admin.go`
- Discovery document issuer must exactly match the expected issuer per OpenID Connect Discovery 1.0, Section 4.3 -> `internal/api/custom_oauth_admin.go`
- Provider count quota is enforced via `config.CustomOAuth.MaxProviders`; exceeding returns `ErrorCodeOverCustomProviderQuota` -> `internal/api/custom_oauth_admin.go`
- `authorization_params` validation blocks 9 reserved OAuth parameters (`client_id`, `client_secret`, `redirect_uri`, `response_type`, `state`, `code_challenge`, `code_challenge_method`, `code_verifier`, `nonce`) -> `internal/api/custom_oauth_admin.go`
- `attribute_mapping` validation blocks 15 protected system fields including `id`, `aud`, `role`, `app_metadata`, `banned_until` -> `internal/api/custom_oauth_admin.go`
- OIDC providers automatically get `"openid"` prepended to their scopes if not already present -> `internal/api/custom_oauth_admin.go`
- All provider URLs are validated via `utilities.ValidateOAuthURL` for SSRF protection -> `internal/api/custom_oauth_admin.go`
- Discovery document fetch has a 10-second timeout and 1MB response size limit -> `internal/api/custom_oauth_admin.go`
- On update, if the OIDC issuer changes, the in-memory OIDC cache is invalidated for both old and new issuer -> `internal/api/custom_oauth_admin.go`
- Deletion hard-removes the provider via `tx.Destroy` and invalidates the OIDC cache if applicable -> `internal/api/custom_oauth_admin.go`
- `PKCEEnabled` defaults to `true`, `Enabled` defaults to `true`, `EmailOptional` defaults to `false` when not specified -> `internal/api/custom_oauth_admin.go`
- `UpdateCustomOAuthProvider` explicitly sets `updated_at = time.Now()` in application code before calling `tx.Update` -> `internal/models/custom_oauth_provider.go`

## Fields

| Column | Type | Lifecycle Role |
|--------|------|---------------|
| id | UUID | PK, generated at creation |
| provider_type | VARCHAR | `"oauth2"` or `"oidc"` |
| identifier | VARCHAR | Unique, must have `"custom:"` prefix |
| name | VARCHAR | Human-readable display name |
| client_id | VARCHAR | OAuth client ID |
| client_secret | VARCHAR | Encrypted OAuth client secret |
| scopes | TEXT[] | OAuth scopes (OIDC always includes `"openid"`) |
| pkce_enabled | BOOLEAN | Whether to use PKCE (default true) |
| attribute_mapping | JSONB | Maps provider claims to user fields |
| authorization_params | JSONB | Extra params for authorization URL |
| enabled | BOOLEAN | Whether provider is active (default true) |
| email_optional | BOOLEAN | Allow sign-in without email (default false) |
| issuer | VARCHAR | OIDC issuer URL (OIDC only) |
| discovery_url | VARCHAR | Custom OIDC discovery URL (OIDC only) |
| authorization_url | VARCHAR | Auth endpoint (OAuth2 only) |
| token_url | VARCHAR | Token endpoint (OAuth2 only) |
| userinfo_url | VARCHAR | Userinfo endpoint (OAuth2 only) |

## Relationships

| Related Entity | Relationship | FK |
|---------------|-------------|-----|
| [[PROC-AUTH-IDENTITIES-LIFECYCLE]] | creates identities | Provider name = `identifier` value |

## Creation Path

1. Admin calls `POST /admin/custom-oauth-providers` with provider configuration.
2. Validation: type-specific required fields, URL SSRF checks, reserved param/field checks, quota check, duplicate identifier check.
3. For OIDC: discovery document is fetched, validated (required fields + issuer match), and cached.
4. Client secret is encrypted.
5. Provider is created in a single transaction.

## Worked Examples

### Query: List all enabled custom providers

```sql
SELECT identifier, name, provider_type, enabled, pkce_enabled
FROM auth.custom_oauth_providers
WHERE enabled = true
ORDER BY created_at DESC;
```

### Enum: Provider types

| provider_type | Required Fields |
|--------------|----------------|
| `oauth2` | `authorization_url`, `token_url`, `userinfo_url` |
| `oidc` | `issuer` (discovery URL derived from issuer or explicit) |

## Built-in Provider Registry

Auth ships with 24 built-in OAuth/OIDC provider integrations, plus support for custom providers. The `Provider()` function in `internal/api/external.go` dispatches by name:

| Provider | Type | OIDC Cache | Source |
|----------|------|------------|--------|
| Apple | OIDC | Yes | `provider/apple.go` |
| Azure | OIDC | Yes | `provider/azure.go` |
| Bitbucket | OAuth2 | No | `provider/bitbucket.go` |
| Discord | OAuth2 | No | `provider/discord.go` |
| Facebook | OAuth2 | No | `provider/facebook.go` |
| Figma | OAuth2 | No | `provider/figma.go` |
| Fly | OAuth2 | No | `provider/fly.go` |
| GitHub | OAuth2 | No | `provider/github.go` |
| GitLab | OAuth2 | No | `provider/gitlab.go` |
| Google | OIDC | Yes | `provider/google.go` |
| Kakao | OAuth2 | No | `provider/kakao.go` |
| Keycloak | OAuth2 | No | `provider/keycloak.go` |
| LinkedIn | OAuth2 | No | `provider/linkedin.go` |
| LinkedIn OIDC | OIDC | Yes | `provider/linkedin_oidc.go` |
| Notion | OAuth2 | No | `provider/notion.go` |
| Snapchat | OAuth2 | No | `provider/snapchat.go` |
| Spotify | OAuth2 | No | `provider/spotify.go` |
| Slack | OAuth2 | No | `provider/slack.go` |
| Slack OIDC | OIDC | No | `provider/slack_oidc.go` |
| Twitch | OAuth2 | No | `provider/twitch.go` |
| Twitter | OAuth1 | No | `provider/twitter.go` |
| X | OAuth2 | No | `provider/x.go` |
| Vercel Marketplace | OIDC | Yes | `provider/vercel_marketplace.go` |
| WorkOS | OAuth2 | No | `provider/workos.go` |
| Zoom | OAuth2 | No | `provider/zoom.go` |
| Custom (`custom:*`) | OAuth2/OIDC | DB-cached | `provider/custom_oauth.go` |

Custom providers (prefixed `custom:`) are loaded from the database via `loadCustomProvider()` and require `config.CustomOAuth.Enabled` to be true. All providers implement the `provider.Provider` interface with `GetOAuthToken()` and `GetUserData()` methods -> `internal/api/external.go`

The OAuth callback flow in `external_oauth.go` handles both OAuth2 (authorization code exchange) and OAuth1 (Twitter) token flows, with PKCE support via `OAuthClientState` records -> `internal/api/external_oauth.go`

## Agent Guidance

- Custom OAuth providers are admin-only configuration; no user-facing CRUD exists. They are managed via the `/admin/custom-oauth-providers` API.
- There is NO audit logging for provider CRUD yet -- this is a documented TODO in the source code. The existing audit system is user-centric and does not support admin infrastructure operations.
- The `identifier` field is the stable reference for the provider throughout the system, including in identity records. It must start with `"custom:"`.
- OIDC discovery documents are cached in the database (`cached_discovery`, `discovery_cached_at`) and in an in-memory cache (`oidcCache`). Cache invalidation happens on issuer change or provider deletion.
- URL validation includes SSRF protection -- private/internal IPs are blocked by `utilities.ValidateOAuthURL`.
- The `enabled` boolean is a soft toggle -- disabling a provider does not delete it or its associated identities, but blocks new authentications.

## Related

- [[SYS-AUTH]] -- parent system artifact
- [[SCH-AUTH]] -- schema definition for custom_oauth_providers table
- [[PROC-AUTH-IDENTITIES-LIFECYCLE]] -- identities created with custom provider names
