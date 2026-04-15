---
id: "PROC-AUTH-WEBAUTHN-CREDENTIALS-LIFECYCLE"
type: process
title: "Auth WebAuthn Credentials (Passkeys) Entity Lifecycle"
domain: supabase-reef
status: draft
last_verified: 2026-04-15
freshness_note: "Derived from GoTrue internal/models/webauthn_credential.go, webauthn_challenge.go, and API passkey handlers; current with source as of verification date."
freshness_triggers:
  - "internal/api/passkey_authentication.go"
  - "internal/api/passkey_manage.go"
  - "internal/api/passkey_registration.go"
  - "internal/models/webauthn_credential.go"
known_unknowns:
  - "Interaction between webauthn_credentials (passkey table) and mfa_factors.web_authn_credential (factor-level WebAuthn) is not clearly documented"
  - "Passkey login flow (passkey_authentication.go) was not fully traced"
tags:
  - auth
  - entity-lifecycle
  - webauthn
  - passkeys
aliases:
  - "webauthn credentials lifecycle"
  - "passkeys lifecycle"
  - "auth.webauthn_credentials lifecycle"
relates_to:
  - type: parent
    target: "[[PROC-AUTH-USERS-LIFECYCLE]]"
  - type: parent
    target: "[[SCH-AUTH]]"
  - type: refines
    target: "[[SYS-AUTH]]"
sources:
  - category: implementation
    type: github
    ref: "internal/api/passkey_manage.go"
    notes: "List, update, delete passkey API handlers"
  - category: implementation
    type: github
    ref: "internal/api/passkey_registration.go"
    notes: "WebAuthn registration options and verification"
  - category: implementation
    type: github
    ref: "internal/models/webauthn_credential.go"
    notes: "WebAuthnCredential model, CRUD, sign count tracking"
  - category: documentation
    type: doc
    ref: "sources/schemas/auth/schema.md"
    notes: "Canonical schema reference for webauthn_credentials table"
---

## Purpose

Documents the lifecycle of rows in the `auth.webauthn_credentials` table. These represent FIDO2/WebAuthn passkeys registered by users for passwordless authentication. Unlike MFA WebAuthn factors (which live in `mfa_factors`), these are standalone passkey credentials managed through the `/passkeys/` API endpoints.

## Key Facts

- `NewWebAuthnCredential` generates a UUID v4 id and maps the go-webauthn library `Credential` struct to the database model -> `internal/models/webauthn_credential.go`
- `credential_id` and `public_key` are stored as `BYTEA` (binary) columns, not exposed in JSON responses -> `internal/models/webauthn_credential.go`
- `transports` is stored as JSONB, serialized from `[]protocol.AuthenticatorTransport` (e.g., `["usb", "nfc", "ble", "internal"]`) -> `internal/models/webauthn_credential.go`
- `backup_eligible` and `backed_up` flags track WebAuthn credential backup state per the spec -> `internal/models/webauthn_credential.go`
- `sign_count` is a monotonic counter updated on each authentication to detect cloned authenticators -> `internal/models/webauthn_credential.go`
- `UpdateLastUsedWithSignCount` updates both `sign_count` and `last_used_at` atomically on each successful authentication -> `internal/models/webauthn_credential.go`
- SSO users cannot register passkeys -- the API returns `ErrorCodeValidationFailed` -> `internal/api/passkey_registration.go`
- Passkey count per user is limited by `config.Passkey.MaxPasskeysPerUser`; exceeding returns `ErrorCodeTooManyPasskeys` -> `internal/api/passkey_registration.go`
- Registration uses a two-step flow: `POST /passkeys/registration/options` (get challenge) then `POST /passkeys/registration/verify` (submit credential) -> `internal/api/passkey_registration.go`
- Existing credentials are added to the exclusion list during registration to prevent duplicate enrollments -> `internal/api/passkey_registration.go`
- `PasskeyUpdate` only allows changing `friendly_name` (max 120 chars) -> `internal/api/passkey_manage.go`
- `PasskeyDelete` hard-deletes the credential via `tx.Destroy` and logs a `PasskeyDeletedAction` audit entry -> `internal/api/passkey_manage.go`
- `DeleteWebAuthnCredentialsByUserID` bulk-deletes all credentials for a user via raw SQL -> `internal/models/webauthn_credential.go`
- `ToWebAuthnCredential()` converts back to the library's Credential type for verification, reconstituting flags and AAGUID -> `internal/models/webauthn_credential.go`
- Challenges for registration are stored in `webauthn_challenges` table with an expiry; expired challenges are cleaned up automatically -> `internal/api/passkey_registration.go`

## Fields

| Column | Type | Lifecycle Role |
|--------|------|---------------|
| id | UUID | PK, generated at creation |
| user_id | UUID | FK to users.id |
| credential_id | BYTEA | WebAuthn credential ID (binary) |
| public_key | BYTEA | Public key for signature verification |
| attestation_type | VARCHAR | Attestation type from registration |
| aaguid | UUID | Authenticator AAGUID (nullable) |
| sign_count | INT | Monotonic counter, updated on each auth |
| transports | JSONB | Supported transports array |
| backup_eligible | BOOLEAN | Whether credential supports backup |
| backed_up | BOOLEAN | Whether credential is currently backed up |
| friendly_name | VARCHAR | User-assigned display name |
| last_used_at | TIMESTAMPTZ | Updated on each authentication |

## Relationships

| Related Entity | Relationship | FK |
|---------------|-------------|-----|
| [[PROC-AUTH-USERS-LIFECYCLE]] | belongs to | `webauthn_credentials.user_id -> users.id` |

## Creation Path

1. Authenticated user calls `POST /passkeys/registration/options`.
2. Server generates WebAuthn creation options with exclusion list and stores a challenge in `webauthn_challenges`.
3. Client interacts with authenticator and calls `POST /passkeys/registration/verify` with the credential response.
4. Server verifies the response, creates `WebAuthnCredential` row, returns passkey metadata.

## Worked Examples

### Query: List all passkeys for a user with usage info

```sql
SELECT id, friendly_name, created_at, last_used_at, sign_count, backup_eligible
FROM auth.webauthn_credentials
WHERE user_id = '550e8400-e29b-41d4-a716-446655440000'
ORDER BY created_at ASC;
```

## Agent Guidance

- There are TWO WebAuthn credential storage mechanisms: `webauthn_credentials` (standalone passkeys) and `mfa_factors.web_authn_credential` (MFA factors). They serve different purposes -- passkeys are for passwordless login, MFA factors are for step-up authentication.
- The `sign_count` field is a cloned authenticator detection mechanism: if the presented sign count is less than or equal to the stored value, the authenticator may have been cloned.
- `credential_id` and `public_key` are binary and not human-readable; always use the UUID `id` for API references.
- Passkey deletion is a hard delete with no soft-delete path. The TODO comment in the source notes that deletion of credentials used by MFA WebAuthn factors should be blocked but is not yet enforced.

## Related

- [[SYS-AUTH]] -- parent system artifact
- [[SCH-AUTH]] -- schema definition for webauthn_credentials table
- [[PROC-AUTH-USERS-LIFECYCLE]] -- parent user entity
- [[PROC-AUTH-MFA-FACTORS-LIFECYCLE]] -- MFA factors can also hold WebAuthn credentials (separate mechanism)
