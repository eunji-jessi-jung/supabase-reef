---
id: "PROC-REALTIME-AUTH"
type: "process"
title: "Realtime Authentication and Authorization Flow"
domain: "supabase-reef"
status: draft
last_verified: 2026-04-15
freshness_note: "Deep scan of authorization.ex, realtime_channel.ex, jwt_verification.ex, user_socket.ex, and channels_authorization.ex"
freshness_triggers:
  - "lib/realtime/tenants/authorization.ex"
  - "lib/realtime/tenants/authorization/policies/"
  - "lib/realtime_web/channels/auth/channels_authorization.ex"
  - "lib/realtime_web/channels/auth/jwt_verification.ex"
  - "lib/realtime_web/channels/realtime_channel.ex"
  - "lib/realtime_web/channels/user_socket.ex"
  - "lib/realtime_web/router.ex"
known_unknowns:
  - "How JWKS keys are refreshed or rotated at runtime"
  - "Whether RLS policy evaluation results are cached between re-auth intervals"
  - "How authorization behaves during a gen_rpc call timeout to a remote node"
tags:
  - realtime
  - auth
  - jwt
  - rls
aliases: []
relates_to:
  - type: "parent"
    target: "[[SYS-REALTIME]]"
  - type: "depends_on"
    target: "[[SYS-AUTH]]"
sources:
  - category: "implementation"
    type: "github"
    ref: "lib/realtime/tenants/authorization.ex"
    notes: "Core RLS policy evaluation logic"
  - category: "implementation"
    type: "github"
    ref: "lib/realtime/tenants/authorization/policies.ex"
    notes: "Policies struct holding broadcast+presence permissions"
  - category: "implementation"
    type: "github"
    ref: "lib/realtime/tenants/authorization/policies/broadcast_policies.ex"
  - category: "implementation"
    type: "github"
    ref: "lib/realtime/tenants/authorization/policies/presence_policies.ex"
  - category: "implementation"
    type: "github"
    ref: "lib/realtime_web/channels/auth/channels_authorization.ex"
    notes: "Token validation entry point with claims check"
  - category: "implementation"
    type: "github"
    ref: "lib/realtime_web/channels/auth/jwt_verification.ex"
    notes: "JWT verification supporting HS, RS, ES, Ed algorithms"
  - category: "implementation"
    type: "github"
    ref: "lib/realtime_web/channels/realtime_channel.ex"
    notes: "Channel join, periodic re-auth, and token refresh handlers"
  - category: "implementation"
    type: "github"
    ref: "lib/realtime_web/channels/user_socket.ex"
    notes: "WebSocket connect-level auth"
  - category: "implementation"
    type: "github"
    ref: "lib/realtime_web/router.ex"
    notes: "REST API auth pipelines"
notes: ""
---

## Purpose

Documents the full authentication and authorization flow for Supabase Realtime, covering JWT validation at WebSocket connect, channel-level RLS policy evaluation, periodic token re-validation, and runtime token refresh. Realtime enforces auth at two layers: connection-level (UserSocket) and channel-level (RealtimeChannel), with RLS policies evaluated against the tenant's Postgres database for private channels.

## Key Facts

- WebSocket `connect/3` in UserSocket validates JWT using tenant's `jwt_secret` (decrypted via `Crypto.decrypt!`) and optional `jwt_jwks`; suspended tenants are rejected immediately → `lib/realtime_web/channels/user_socket.ex`
- `authorize_conn/3` requires both `role` and `exp` claims present in JWT; missing either returns `{:error, :missing_claims}` → `lib/realtime_web/channels/auth/channels_authorization.ex`
- JWT verification supports 4 algorithm families: HS (256/384/512), RS (256/384/512), ES (256/384/512), and Ed (25519/448) via Joken signers → `lib/realtime_web/channels/auth/jwt_verification.ex`
- JWKS lookup matches by `kid` header and `kty` field (RSA, EC, OKP, oct); for HS algorithms with JWKS, falls back to `jwt_secret` if no matching `kid` found → `lib/realtime_web/channels/auth/jwt_verification.ex`
- Token re-validation fires every 5 minutes (`@confirm_token_ms_interval`) or at token expiry time, whichever is sooner; expired tokens during re-auth trigger `shutdown_response` terminating the channel → `lib/realtime_web/channels/realtime_channel.ex`
- Client can send an `access_token` message to refresh the JWT mid-session; this resets policies to `nil`, re-validates the new token, re-evaluates RLS policies, and re-subscribes to Postgres CDC if needed → `lib/realtime_web/channels/realtime_channel.ex`
- Tokens prefixed with `sb_` are treated as service-role tokens and bypass client token assignment, using the `tenant_token` from socket assigns instead → `lib/realtime_web/channels/realtime_channel.ex`
- RLS policy evaluation runs inside a Postgres transaction that inserts test messages into `realtime.messages`, checks if the row is readable (via `SELECT` with RLS), then issues `ROLLBACK AND CHAIN` to avoid persisting test data → `lib/realtime/tenants/authorization.ex`
- Write policy evaluation attempts an `INSERT` per extension; `insufficient_privilege` Postgres error means write denied, successful insert means write allowed → `lib/realtime/tenants/authorization.ex`
- Policies struct tracks read/write booleans independently for both broadcast and presence extensions; `nil` means not yet evaluated → `lib/realtime/tenants/authorization/policies.ex`
- Authorization checks are rate-limited per tenant via `GenCounter`; when rate limit triggers, returns `{:error, :increase_connection_pool}` instead of hitting the database → `lib/realtime/tenants/authorization.ex`
- For cross-node authorization, `GenRpc.call` routes the policy check to the node owning the database connection, with `tenant_id` as the consistent routing key → `lib/realtime/tenants/authorization.ex`
- REST admin API uses `check_auth` plug validating Bearer token against `api_jwt_secret` with a token blocklist; tenant-scoped endpoints use `AssignTenant` plug for tenant resolution → `lib/realtime_web/router.ex`
- `set_conn_config/2` sets 6 Postgres session variables (role, realtime.topic, request.jwt.claims, request.jwt.claim.sub, request.jwt.claim.role, request.headers) enabling RLS policies to reference JWT claims → `lib/realtime/tenants/authorization.ex`
- Private channels (`private?: true`) require RLS read authorization on join; if broadcast read is `false`, the join returns `{:error, :unauthorized}` with the topic name in the message → `lib/realtime_web/channels/realtime_channel.ex`

## Steps

1. **WebSocket connect** -- client provides `apikey` param or `x-api-key` header; `UserSocket.connect/3` resolves tenant from host, validates JWT against tenant's secret/JWKS, checks tenant is not suspended, and enforces connection rate limits
2. **Channel join** -- client sends `phx_join` to `realtime:<topic>` with optional `access_token`; channel resolves tenant from cache, enforces user/join/channel limits, and calls `confirm_token` for JWT validation
3. **Token validation** -- `ChannelsAuthorization.authorize_conn/3` cleans the token (strips whitespace, URI-decodes), verifies structure via `Joken.peek_claims`, selects signer based on header algorithm and JWKS/secret, then validates claims including `exp`
4. **Authorization context build** -- `assign_authorization_context/3` constructs an `Authorization` struct with `tenant_id`, `topic`, `headers`, `claims`, `role`, and `sub` from validated JWT
5. **RLS policy evaluation (private channels)** -- `maybe_assign_policies/3` calls `get_read_authorizations`, which opens a DB transaction, sets connection config with JWT claims, inserts test messages per extension, checks if they're readable via RLS, then rolls back
6. **Subscription active** -- on successful auth and policy check, channel subscribes to PubSub topics, sets up Postgres CDC subscriptions, and starts presence sync if enabled
7. **Periodic re-auth** -- `:confirm_token` message fires at `min(5 minutes, token_ttl)`; on failure, channel is terminated via `shutdown_response`
8. **Runtime token refresh** -- client sends `access_token` event with new JWT; channel resets policies, re-validates, re-evaluates RLS, and re-subscribes to CDC

## Worked Examples

### Private Channel Join with RLS

A client joins `realtime:private-room` on a private channel. The flow:
1. `UserSocket.connect` validates the initial JWT and assigns claims
2. `RealtimeChannel.join` checks limits, calls `confirm_token`
3. Since `private?: true`, `maybe_assign_policies` is invoked
4. `Authorization.get_read_authorizations` opens a transaction, calls `set_conn_config` to set `role` and `request.jwt.claims` as Postgres session vars
5. Test messages are inserted for `broadcast` and `presence` extensions
6. A `SELECT` query checks which inserted messages are visible under RLS -- visible means read allowed
7. Transaction is rolled back via `ROLLBACK AND CHAIN`
8. If broadcast read is `false`, join fails with `{:error, :unauthorized}`

### Token Refresh Mid-Session

1. Client sends `{"event": "access_token", "payload": {"access_token": "<new-jwt>"}}`
2. Channel assigns new token, resets `policies` to `nil`
3. `confirm_token` validates the new JWT; on success, schedules next re-auth
4. `assign_authorization_context` rebuilds context with new claims
5. `maybe_assign_policies` re-evaluates RLS with the new role/claims
6. Postgres CDC params are updated with new claims and re-subscribed

## Related

- [[SYS-REALTIME]] -- parent system
- [[SYS-AUTH]] -- issues the JWTs that Realtime validates
