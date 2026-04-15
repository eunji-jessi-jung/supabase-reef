---
id: "RISK-AUTH-KNOWN-GAPS"
type: "risk"
title: "Auth Known Risks, Gaps, and Backlog Candidates"
domain: "supabase-reef"
status: draft
last_verified: 2026-04-15
freshness_note: "Deep code scan of all TODO/FIXME/HACK/BUG markers across internal/ non-test files"
freshness_triggers:
  - "internal/api/"
  - "internal/hooks/"
  - "internal/mailer/"
  - "internal/models/"
  - "internal/utilities/"
known_unknowns:
  - "Severity assessments are based on signal density and potential impact, not production incident data"
  - "Some TODOs may be resolved but not removed from code"
  - "SAML abuse detection TODO may have mitigations outside the codebase (e.g., WAF rules)"
tags:
  - risk-catalog
aliases: []
relates_to:
  - type: "parent"
    target: "[[SYS-AUTH]]"
  - type: "refines"
    target: "[[PROC-AUTH-ERROR-HANDLING]]"
sources:
  - category: "implementation"
    type: "github"
    ref: "internal/api/admin.go"
  - category: "implementation"
    type: "github"
    ref: "internal/api/custom_oauth_admin.go"
  - category: "implementation"
    type: "github"
    ref: "internal/api/external.go"
  - category: "implementation"
    type: "github"
    ref: "internal/api/jwks.go"
  - category: "implementation"
    type: "github"
    ref: "internal/api/mail.go"
  - category: "implementation"
    type: "github"
    ref: "internal/api/passkey_manage.go"
  - category: "implementation"
    type: "github"
    ref: "internal/api/phone.go"
  - category: "implementation"
    type: "github"
    ref: "internal/api/samlacs.go"
  - category: "implementation"
    type: "github"
    ref: "internal/api/signup.go"
  - category: "implementation"
    type: "github"
    ref: "internal/api/user.go"
  - category: "implementation"
    type: "github"
    ref: "internal/api/verify.go"
  - category: "implementation"
    type: "github"
    ref: "internal/hooks/hookserrors/hookserrors.go"
  - category: "implementation"
    type: "github"
    ref: "internal/hooks/hookshttp/hookshttp.go"
  - category: "implementation"
    type: "github"
    ref: "internal/models/audit_log_entry.go"
  - category: "implementation"
    type: "github"
    ref: "internal/models/sessions.go"
notes: ""
---

## Description

Catalogs risk signals found in the Auth codebase during deep scanning. A full grep of TODO/FIXME/HACK/BUG markers across non-test Go files yielded 35+ signals across 25+ files, grouped into 7 themes.

## Key Facts

- 35+ risk signals found across 25+ files in internal/, with highest density in custom_oauth_admin.go (5 signals) and mail.go (3 signals) → `internal/`
- Missing audit logging: 5 TODOs in custom OAuth provider admin operations (create, update, delete) for infrastructure-level changes that should be tracked → `internal/api/custom_oauth_admin.go`
- Deprecation debt: at least 6 TODOs across verify.go, phone.go, signup.go, admin.go, mail.go, and provider.go mark behaviors that should be deprecated but remain active → `internal/api/verify.go`, `internal/api/phone.go`, `internal/api/signup.go`
- Hook error handling defaults to HTTP 500 when hook returns no http_code, acknowledged as needing to be 400 (BadRequest); changing it would be a BC break → `internal/hooks/hookserrors/hookserrors.go`
- Asymmetric key handling in HTTP hooks is incomplete, blocked on library upgrade → `internal/hooks/hookshttp/hookshttp.go:266`
- Passkey management operations are not restricted for credentials used as MFA WebAuthn factors, potentially allowing deletion of MFA credentials → `internal/api/passkey_manage.go:29`
- SSO user detection relies on a flag (`is_sso_user`) instead of checking the identities table, creating a data integrity risk → `internal/api/user.go:120`
- SAML RelayState lacks abuse detection binding (no HTTP-Only cookie binding to prevent cross-site relay state reuse) → `internal/api/samlacs.go:98`
- Audit log bug: multiple audit events in the same request overwrite each other via `LogEntrySetFields`; a workaround using immediate separate log entries was added → `internal/models/audit_log_entry.go:134`
- Email change token/hash fields are mismatched when secure email change is disabled, producing `TokenHashNew = Hash(CurEmail, Token)` instead of the expected mapping → `internal/api/mail.go:813`
- Session timestamp stored without timezone in database, requiring manual UTC conversion before updates → `internal/models/sessions.go:156`
- Rate limit bypass on autoconfirm: both phone and email flows skip rate limiting when autoconfirm is enabled, marked for deprecation → `internal/api/phone.go:88`, `internal/api/mail.go:796`
- OIDC discovery endpoint hardcodes supported signing algorithms instead of deriving from actual key configuration → `internal/api/jwks.go:87`
- OAuth error details duplicated in both query params and URL fragment on redirects, with fragment portion marked for deprecation → `internal/api/external.go:796`

## Findings

| # | Location | Signal | Theme | Severity |
|---|----------|--------|-------|----------|
| 1 | `internal/api/custom_oauth_admin.go:22` | TODO: Admin Audit Logging for Custom OAuth/OIDC Providers | Audit gaps | medium |
| 2 | `internal/api/custom_oauth_admin.go:232` | TODO: Implement proper admin audit logging for infrastructure changes | Audit gaps | medium |
| 3 | `internal/api/custom_oauth_admin.go:328` | TODO: Add admin audit logging here | Audit gaps | medium |
| 4 | `internal/api/custom_oauth_admin.go:381` | TODO: Add admin audit logging here | Audit gaps | medium |
| 5 | `internal/api/custom_oauth_admin.go:44` | All create/update/delete operations lack audit logging | Audit gaps | medium |
| 6 | `internal/hooks/hookserrors/hookserrors.go:69` | Changing error codes would be a BC break | Error handling | medium |
| 7 | `internal/hooks/hookserrors/hookserrors.go:78` | Should be BadRequest as default, not 500 | Error handling | medium |
| 8 | `internal/hooks/hookshttp/hookshttp.go:266` | Handle asymmetric case once library upgraded | Incomplete implementation | medium |
| 9 | `internal/api/verify.go:61` | Deprecate token query param from GET /verify | Deprecation debt | medium |
| 10 | `internal/api/phone.go:88` | Deprecate: rate limits should still be applied to autoconfirm | Deprecation debt | medium |
| 11 | `internal/api/mail.go:796` | Deprecate: rate limits should still be applied to autoconfirm | Deprecation debt | medium |
| 12 | `internal/api/signup.go:100` | Deprecate "provider" field | Deprecation debt | low |
| 13 | `internal/api/admin.go:410` | Deprecate "provider" field | Deprecation debt | low |
| 14 | `internal/api/external.go:796` | Deprecate returning error details in query fragment | Deprecation debt | low |
| 15 | `internal/api/passkey_manage.go:29` | Should not allow operations on credentials used for MFA | Security | high |
| 16 | `internal/api/user.go:120` | Check SSO via identities table, not via flag | Data integrity | medium |
| 17 | `internal/api/samlacs.go:98` | Add abuse detection to bind RelayState with HTTP-Only cookie | Security | medium |
| 18 | `internal/models/audit_log_entry.go:134` | BUG: Multiple audit events overwrite each other in same request | Data integrity | medium |
| 19 | `internal/api/mail.go:813` | BUG: Token/hash field mismatch during non-secure email change | Correctness | high |
| 20 | `internal/api/mail.go:844` | BUG: Matches current behavior but is not intuitive | Correctness | medium |
| 21 | `internal/models/sessions.go:156` | Timestamp without timezone requires manual UTC conversion | Data integrity | low |
| 22 | `internal/api/jwks.go:87` | Signing algorithms hardcoded instead of derived from config | Configuration drift | low |
| 23 | `internal/models/linking.go:143` | TODO: Backfill missing identity for the user | Data integrity | medium |

## Themes

### Missing Audit Logging (5 signals, medium)

Custom OAuth provider admin operations (create, update, delete) lack audit logging. These are infrastructure-level changes that should be tracked for security compliance.

**Impact if unaddressed:** No audit trail for OAuth provider configuration changes, making security incident investigation harder.

### Deprecation Debt (6+ signals, medium)

Multiple API behaviors are marked for deprecation but remain active: token query param on GET /verify, rate limit bypass on autoconfirm (phone and email), legacy "provider" field in signup/admin, and error details in OAuth redirect fragments.

**Impact if unaddressed:** Deprecated behaviors accumulate, making breaking changes harder and increasing the API surface to secure.

### Error Handling (2 signals, medium)

Hooks error system returns 500 errors where 400 would be more appropriate. Changing this is acknowledged as a backward compatibility break. Empty-message errors are silently ignored regardless of HTTP code.

**Impact if unaddressed:** Clients may misinterpret server errors as transient failures rather than client errors, leading to inappropriate retry behavior.

### Security Gaps (2 signals, medium-high)

Passkey management endpoints do not restrict operations on credentials used as MFA WebAuthn factors. SAML RelayState lacks HTTP-Only cookie binding for abuse detection.

**Impact if unaddressed:** MFA credentials could be deleted through passkey management API. SAML relay state could be reused in cross-site attacks.

### Correctness Bugs (2 signals, high)

Email change flow has a documented token/hash field mismatch when secure email change is disabled. The bug causes `TokenHashNew` to contain the hash of the current email's token instead of the new email's token.

**Impact if unaddressed:** Email change verification may silently succeed or fail in unexpected ways; the mismatch is documented but unfixed.

### Data Integrity (3 signals, medium)

SSO detection uses a column flag rather than the identities table. Session timestamps stored without timezone. Missing identity backfill for edge-case account linking.

**Impact if unaddressed:** SSO status may be inaccurate if the flag drifts from actual identity data. Timezone-naive timestamps could cause session validity miscalculation across regions.

### Incomplete Implementation (1 signal, medium)

HTTP hook dispatcher cannot handle asymmetric key verification, blocked on a library upgrade.

**Impact if unaddressed:** Hooks using asymmetric signing cannot be verified, limiting hook security options.

## Related

- [[SYS-AUTH]] -- parent system
- [[PROC-AUTH-ERROR-HANDLING]] -- error handling patterns that several of these risks relate to
