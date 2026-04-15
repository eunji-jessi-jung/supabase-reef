---
id: "CON-AUTH-CLIENT-SDK"
type: "contract"
title: "Auth to Client SDK Contract"
domain: "supabase-reef"
status: "draft"
last_verified: 2026-04-15
freshness_note: "snorkel-depth scan"
freshness_triggers:
  - "internal/api/api.go"
  - "packages/core/auth-js/src/GoTrueClient.ts"
known_unknowns:
  - "Complete API contract between auth-js and Auth server"
  - "How auth-js handles version skew between SDK and server"
  - "Error response format contract"
tags:
  - contract
  - rest-api
aliases: []
relates_to:
  - type: "integrates_with"
    target: "[[SYS-AUTH]]"
  - type: "integrates_with"
    target: "[[SYS-CLIENT-SDK]]"
sources:
  - category: "implementation"
    type: "github"
    ref: "packages/core/auth-js/src/GoTrueClient.ts"
notes: ""
---

## Parties

- **Producer**: Auth (REST API for auth operations)
- **Consumer**: Client SDK (auth-js wraps Auth API)

## Key Facts

- auth-js calls Auth REST endpoints for signup, login, token refresh, logout, user management → `packages/core/auth-js/src/GoTrueClient.ts`
- Auth-js manages session lifecycle: stores tokens, auto-refreshes, fires state change events → `packages/core/auth-js/src/`
- fetchWithAuth injects auth tokens into all other sub-client requests → `packages/core/supabase-js/src/lib/fetch.ts`
- Auth-js still references "GoTrue" in class names (legacy naming) → `packages/core/auth-js/src/GoTrueClient.ts`
- Scripts include `publish-gotrue-legacy.ts` suggesting a migration from the old package name → `scripts/publish-gotrue-legacy.ts`

## Agreement

auth-js is the primary client for Auth's REST API. It calls endpoints defined in Auth's router (signup, token, verify, recover, user, factors) and expects the documented JSON response shapes. Token refresh depends on the refresh_token grant type working consistently.

## Current State

The contract is version-coupled — auth-js and Auth server evolve together. The SDK's canary release model means new SDK versions may hit production before the corresponding server version is deployed.

## Related

- [[SYS-AUTH]] -- API producer
- [[SYS-CLIENT-SDK]] -- API consumer
- [[API-AUTH]] -- server-side API
- [[API-CLIENT-SDK]] -- client-side API
