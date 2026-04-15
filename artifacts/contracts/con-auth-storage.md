---
id: "CON-AUTH-STORAGE"
type: "contract"
title: "Auth to Storage JWT Contract"
domain: "supabase-reef"
status: "draft"
last_verified: 2026-04-15
freshness_note: "snorkel-depth scan"
freshness_triggers:
  - "internal/tokens/"
  - "src/internal/auth/"
known_unknowns:
  - "Exact JWT claims that Storage relies on beyond role and sub"
  - "How Storage handles expired or revoked tokens"
  - "Whether Storage validates tokens independently or trusts a gateway"
tags:
  - contract
  - jwt
  - rls
aliases: []
relates_to:
  - type: "integrates_with"
    target: "[[SYS-AUTH]]"
  - type: "integrates_with"
    target: "[[SYS-STORAGE]]"
sources:
  - category: "implementation"
    type: "github"
    ref: "src/internal/auth/index.ts"
notes: ""
---

## Parties

- **Producer**: Auth (issues JWTs with user identity and role claims)
- **Consumer**: Storage (validates JWTs and sets claims on Postgres connection for RLS)

## Key Facts

- Auth issues JWTs signed with a shared secret or asymmetric key → `internal/tokens/`
- Storage validates JWTs using the same signing key and extracts claims → `src/internal/auth/jwt.ts`
- JWT claims set on Postgres connection enable RLS policy evaluation → architectural pattern
- service_role JWT bypasses RLS for admin operations → common Supabase pattern
- JWKS endpoint available for key discovery → `internal/api/jwks.go`

## Agreement

Auth produces JWTs containing at minimum: `sub` (user ID), `role` (e.g., authenticated, service_role), and `aud`. Storage validates these tokens and uses the claims to set Postgres session variables that RLS policies reference. Both services must share the same JWT signing secret.

## Current State

This contract is implicit — there is no formal schema or versioning for JWT claims. Changes to Auth's JWT structure could break Storage's RLS evaluation. The JWKS endpoint provides a discovery mechanism for key rotation.

## Related

- [[SYS-AUTH]] -- JWT producer
- [[SYS-STORAGE]] -- JWT consumer
- [[PROC-AUTH-AUTH]] -- how Auth issues tokens
- [[PROC-STORAGE-AUTH]] -- how Storage validates tokens
