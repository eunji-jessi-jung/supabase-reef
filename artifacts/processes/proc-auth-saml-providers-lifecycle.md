---
id: "PROC-AUTH-SAML-PROVIDERS-LIFECYCLE"
type: process
title: "Auth SAML Providers Entity Lifecycle"
domain: supabase-reef
status: draft
last_verified: 2026-04-15
freshness_note: "Derived from GoTrue internal/models/sso.go and internal/api/ssoadmin.go; current with source as of verification date."
freshness_triggers:
  - "internal/api/ssoadmin.go"
  - "internal/models/sso.go"
known_unknowns:
  - "SAML provider update flow for domain changes is complex; full cascade behavior not fully traced"
  - "Relationship between saml_providers and sso_providers is 1:1 but modeled as has_one -- unclear if multiple SAML providers per SSO provider is planned"
tags:
  - auth
  - entity-lifecycle
  - saml
  - sso
aliases:
  - "saml providers lifecycle"
  - "auth.saml_providers lifecycle"
  - "sso providers lifecycle"
relates_to:
  - type: parent
    target: "[[SCH-AUTH]]"
  - type: refines
    target: "[[SYS-AUTH]]"
sources:
  - category: implementation
    type: github
    ref: "internal/api/ssoadmin.go"
    notes: "Admin CRUD for SSO/SAML providers including metadata validation"
  - category: implementation
    type: github
    ref: "internal/models/sso.go"
    notes: "SSOProvider, SAMLProvider, SSODomain, SAMLRelayState models and queries"
  - category: documentation
    type: doc
    ref: "sources/schemas/auth/schema.md"
    notes: "Canonical schema reference for saml_providers, sso_providers, sso_domains tables"
---

## Purpose

Documents the lifecycle of SAML provider configuration entities, which span three related tables: `sso_providers` (parent), `saml_providers` (SAML-specific configuration), and `sso_domains` (email domain mappings). These are admin-managed configuration entities that define how SSO/SAML identity providers are integrated into the auth system.

## Key Facts

- `SSOProvider` is the parent entity with `has_one` relationship to `SAMLProvider` and `has_many` to `SSODomain` -> `internal/models/sso.go`
- `SSOProvider.Type()` always returns `"saml"` -- currently the only supported SSO provider type -> `internal/models/sso.go`
- `SSOProvider.IsEnabled()` returns true if `Disabled` is nil or false; disabled providers block authentication -> `internal/models/sso.go`
- Provider creation requires either `metadata_xml` or `metadata_url` (not both); metadata is parsed and validated for EntityID and exactly one IDPSSODescriptor -> `internal/api/ssoadmin.go`
- `metadata_url` must use HTTPS scheme; HTTP is rejected during validation -> `internal/api/ssoadmin.go`
- SAML metadata must contain valid UTF-8 and exactly one `IDPSSODescriptor`; zero or multiple descriptors are rejected -> `internal/api/ssoadmin.go`
- Duplicate EntityID detection: `FindSAMLProviderByEntityID` checks for existing providers before creation -> `internal/api/ssoadmin.go`
- Domain uniqueness is enforced: each domain can only be assigned to one SSO provider, checked via `FindSSOProviderByDomain` -> `internal/api/ssoadmin.go`
- `resource_id` is an optional external identifier that supports prefix-based filtering via `FindAllSSOProvidersByFilter` -> `internal/models/sso.go`
- Provider lookup supports three methods: by UUID `id`, by `resource_id` (with `resource_` prefix in URL), or by domain email address -> `internal/models/sso.go`
- `NameIDFormat` supports four SAML values: persistent, emailAddress, transient, unspecified -> `internal/api/ssoadmin.go`
- `SAMLAttributeMapping` maps SAML assertion attributes to user identity data; supports single name, multiple names, defaults, and array mode -> `internal/models/sso.go`
- `SAMLRelayState` is a transient entity (cleaned up after 24h) that tracks the redirect_to URL during SSO flows -> `internal/models/sso.go`
- Provider updates enforce EntityID immutability: the new metadata's EntityID must match the existing provider's EntityID -> `internal/api/ssoadmin.go`
- All SSO provider queries use `tx.Eager()` to eagerly load the SAMLProvider and SSODomains associations -> `internal/models/sso.go`

## Fields: sso_providers

| Column | Type | Lifecycle Role |
|--------|------|---------------|
| id | UUID | PK |
| resource_id | VARCHAR | Optional external identifier |
| disabled | BOOLEAN | When true, blocks SSO authentication |
| created_at | TIMESTAMPTZ | Set at creation |
| updated_at | TIMESTAMPTZ | Updated on modification |

## Fields: saml_providers

| Column | Type | Lifecycle Role |
|--------|------|---------------|
| id | UUID | PK |
| sso_provider_id | UUID | FK to sso_providers.id |
| entity_id | VARCHAR | SAML EntityID from metadata (immutable after creation) |
| metadata_xml | TEXT | Full SAML metadata XML |
| metadata_url | VARCHAR | URL to fetch metadata (optional) |
| attribute_mapping | JSONB | Maps SAML attributes to user fields |
| name_id_format | VARCHAR | SAML NameID format |

## Relationships

| Related Entity | Relationship | FK |
|---------------|-------------|-----|
| sso_providers | parent of saml_providers | `saml_providers.sso_provider_id` |
| sso_providers | parent of sso_domains | `sso_domains.sso_provider_id` |
| [[PROC-AUTH-IDENTITIES-LIFECYCLE]] | creates identities | Provider prefix `sso:{uuid}` |

## Creation Path

1. Admin calls `POST /admin/sso/providers` with `type: "saml"` and metadata (URL or XML).
2. Metadata is fetched (if URL) and parsed for EntityID and IDPSSODescriptor.
3. Duplicate EntityID check is performed.
4. Domain uniqueness is validated for each domain.
5. `SSOProvider`, `SAMLProvider`, and `SSODomain` records are created in a single eager transaction.

## Worked Examples

### Query: Find the SSO provider for an email domain

```sql
SELECT sp.id, saml.entity_id, sp.disabled
FROM auth.sso_providers sp
JOIN auth.saml_providers saml ON saml.sso_provider_id = sp.id
JOIN auth.sso_domains sd ON sd.sso_provider_id = sp.id
WHERE sd.domain = 'example.com';
```

### Enum: Supported NameID formats

| Format | Value |
|--------|-------|
| Persistent | `urn:oasis:names:tc:SAML:2.0:nameid-format:persistent` |
| Email Address | `urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress` |
| Transient | `urn:oasis:names:tc:SAML:2.0:nameid-format:transient` |
| Unspecified | `urn:oasis:names:tc:SAML:1.1:nameid-format:unspecified` |

## Agent Guidance

- SAML providers are a three-table composite (sso_providers + saml_providers + sso_domains). Always use eager loading when querying.
- The EntityID is immutable after creation -- metadata updates must preserve the same EntityID.
- Domain lookup is the primary method for routing SSO logins: the email's domain is matched against `sso_domains.domain`.
- The `resource_id` field is for external system integration (e.g., linking to a Terraform resource). It supports prefix-based filtering but is not the primary key.
- `SAMLRelayState` entries are ephemeral and cleaned up after 24 hours; they are not configuration but runtime state for the SSO flow.

## Related

- [[SYS-AUTH]] -- parent system artifact
- [[SCH-AUTH]] -- schema definition for saml_providers, sso_providers, sso_domains tables
- [[PROC-AUTH-IDENTITIES-LIFECYCLE]] -- SSO creates identities with `sso:` prefix provider
