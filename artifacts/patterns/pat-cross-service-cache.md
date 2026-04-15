---
id: "PAT-CROSS-SERVICE-CACHE"
type: "pattern"
title: "Cross-Service Caching Pattern"
domain: "supabase-reef"
status: draft
last_verified: 2026-04-15
freshness_note: "scuba-depth cross-service pattern analysis"
freshness_triggers:
  - "src/internal/cache/lru.ts"
  - "src/internal/cache/ttl.ts"
  - "src/internal/cache/monitoring.ts"
  - "src/internal/auth/jwt.ts"
  - "internal/api/provider/oidc_cache.go"
  - "internal/utilities/hibpcache.go"
  - "lib/realtime/tenants/cache.ex"
  - "lib/realtime/tenants/connect/get_tenant.ex"
  - "apps/studio/data/query-client.ts"
  - "packages/core/auth-js/src/lib/local-storage.ts"
known_unknowns:
  - "Full list of named LRU/TTL caches in Storage beyond JWT cache"
  - "Whether Cachex in Realtime uses distributed caching across nodes or is node-local only"
  - "Studio's React Query staleTime for storage-specific queries"
tags:
  - pattern
  - cross-service
  - cache
  - performance
aliases:
  - "Caching Strategy Pattern"
relates_to:
  - type: "related"
    target: "[[PROC-AUTH-AUTH]]"
  - type: "related"
    target: "[[PROC-STORAGE-AUTH]]"
  - type: "related"
    target: "[[PROC-REALTIME-AUTH]]"
  - type: "related"
    target: "[[PROC-STUDIO-AUTH]]"
  - type: "related"
    target: "[[PROC-CLIENT-SDK-AUTH]]"
sources:
  - category: "implementation"
    type: "github"
    ref: "src/internal/cache/lru.ts"
  - category: "implementation"
    type: "github"
    ref: "src/internal/cache/ttl.ts"
  - category: "implementation"
    type: "github"
    ref: "src/internal/cache/monitoring.ts"
  - category: "implementation"
    type: "github"
    ref: "src/internal/auth/jwt.ts"
  - category: "implementation"
    type: "github"
    ref: "internal/api/provider/oidc_cache.go"
  - category: "implementation"
    type: "github"
    ref: "internal/utilities/hibpcache.go"
  - category: "implementation"
    type: "github"
    ref: "lib/realtime/tenants/cache.ex"
  - category: "implementation"
    type: "github"
    ref: "apps/studio/data/query-client.ts"
  - category: "implementation"
    type: "github"
    ref: "packages/core/auth-js/src/lib/local-storage.ts"
notes: ""
---

## Overview

Each Supabase service implements caching differently, reflecting its runtime environment, data access patterns, and consistency requirements. Auth uses purpose-built in-memory caches (OIDC provider cache, Bloom filter for HIBP). Storage has a comprehensive cache framework with LRU and TTL implementations, OpenTelemetry monitoring, and size-bounded eviction. Realtime uses Cachex (Elixir library) with distributed invalidation. Studio relies on React Query's client-side cache. The Client SDK uses browser localStorage or in-memory adapters for session persistence.

## Key Facts

- Auth's `OIDCProviderCache` uses a `sync.RWMutex`-protected map with TTL-based expiry and `singleflight.Group` to deduplicate concurrent fetches for the same OIDC issuer → `internal/api/provider/oidc_cache.go`
- Auth's OIDC cache serves stale entries on fetch failure, prioritizing availability during transient network issues → `internal/api/provider/oidc_cache.go`
- Auth's `HIBPBloomCache` uses a Bloom filter (`bits-and-blooms/bloom/v3`) for password breach checking, clearing itself at 80% capacity to maintain low false positive rates → `internal/utilities/hibpcache.go`
- Storage's cache framework provides two implementations: `LruCache` (based on `lru-cache` npm package, max items + max size in bytes) and `TtlCache` (based on `@isaacs/ttlcache`, time-based expiry) → `src/internal/cache/lru.ts` and `src/internal/cache/ttl.ts`
- Storage's `createLruCache` and `createTtlCache` factory functions wrap caches with `MonitoredCache` that reports OpenTelemetry metrics: `cache_entries`, `cache_size_bytes`, `cache_requests_total` (with hit/miss/stale outcome), and `cache_evictions_total` → `src/internal/cache/monitoring.ts`
- Storage's JWT cache specifically uses LRU with max 65536 items, 50 MiB max size, 5s TTL resolution, and 1-minute purge stale interval → `src/internal/auth/jwt.ts`
- Storage JWT cache keys are SHA-256 of `token + secret + JWKS_fingerprint`, where the JWKS fingerprint itself is cached in a `WeakMap` to avoid repeated hashing → `src/internal/auth/jwt.ts`
- Realtime uses Cachex with configurable `tenant_cache_expiration` TTL for tenant lookups; invalidation is distributed across cluster nodes via `GenRpc.multicast` → `lib/realtime/tenants/cache.ex`
- Realtime supports both local invalidation (`invalidate_tenant_cache`) and global update (`global_cache_update`) for tenant config changes → `lib/realtime/tenants/cache.ex`
- Studio's React Query client uses a default `staleTime` of 60 seconds (1 minute) and max 3 retries with exponential backoff (doubling from 1s, capped at 30s) → `apps/studio/data/query-client.ts`
- Studio respects `Retry-After` headers from rate-limited responses (429), using the header value as the retry delay instead of exponential backoff → `apps/studio/data/query-client.ts`
- Client SDK stores sessions in localStorage with key `sb-{project-ref}-auth-token`; for non-browser environments, `memoryLocalStorageAdapter` provides an in-memory fallback → `packages/core/auth-js/src/lib/local-storage.ts`
- Storage's `MonitoredCache` tracks evictions but only for capacity-pressure evictions (`reason === 'evict'`), excluding TTL expiry and manual removal from eviction metrics → `src/internal/cache/monitoring.ts`

## Where It Appears

| Aspect | Auth (Go) | Storage (TS) | Realtime (Elixir) | Studio (React) | Client SDK (TS) |
|--------|-----------|-------------|-------------------|----------------|-----------------|
| **Cache type** | In-memory map + Bloom filter | LRU + TTL (npm libs) | Cachex (Elixir) | React Query | localStorage / memory |
| **What's cached** | OIDC providers, HIBP hashes | JWT payloads, tenant config | Tenant records | API responses | Auth sessions |
| **TTL mechanism** | Time check on access | lru-cache ttl option | Cachex expiration | staleTime (60s default) | Token expiry |
| **Size bound** | None (map), capacity-based (Bloom) | Max items + max bytes | Cachex default | None (GC) | Single entry |
| **Invalidation** | `Invalidate(issuer)`, auto-clear at 80% | LRU eviction, PubSub | `GenRpc.multicast` cluster-wide | React Query refetch | Session remove on sign-out |
| **Monitoring** | None | OpenTelemetry (hits, misses, evictions, size) | None visible | None | None |
| **Stale-while-revalidate** | Yes (OIDC cache) | No | No | Yes (React Query default) | No |
| **Concurrency control** | singleflight + RWMutex | LRU lib internal | Cachex internal | React Query | Navigator Lock API |

## Design Intent

Each service caches what matters most for its hot path. Auth caches OIDC discovery documents (expensive external HTTP calls) and HIBP hashes (high-volume lookups). Storage caches JWT verification results (called on every request). Realtime caches tenant config (needed on every WebSocket connect). Studio caches API responses (reduces dashboard latency). The lack of a shared caching layer is intentional: each service's cache has different consistency, size, and invalidation requirements.

## Trade-offs

- **No shared cache infrastructure**: Each service manages its own caches, avoiding coordination overhead but preventing cache sharing (e.g., Storage and Realtime both resolve tenant config independently).
- **Storage's observability advantage**: Storage is the only service with comprehensive cache metrics via OpenTelemetry, making it easier to diagnose cache-related performance issues.
- **Auth's stale-on-error**: OIDC cache serves stale data on network failure, which is a deliberate availability-over-consistency choice that could theoretically serve outdated provider config.
- **Studio's React Query**: Provides stale-while-revalidate semantics by default, which is good for UX but means dashboard data can be up to 60 seconds stale.

## Agent Guidance

- Storage's JWT cache is the most performance-critical cache in the system. If JWT verification is slow, check the cache hit rate via the `cache_requests_total` metric with `outcome` label.
- Realtime tenant cache invalidation is cluster-wide via `GenRpc.multicast`. If a tenant config change doesn't take effect, check whether the multicast reached all nodes.
- Auth's OIDC cache TTL is configurable via `globalConfig.External.OIDCProviderCacheTTL`. If an OIDC provider changes its configuration, the cache must expire or be manually invalidated.
- Studio's query cache skip-retry list (`SKIP_RETRY_PATHNAME_MATCHERS`) prevents retries on known-expensive endpoints like `/run-lints` and `/analytics/endpoints/logs.all`.

## Related

- [[PROC-AUTH-AUTH]] -- Auth authentication uses OIDC cache
- [[PROC-STORAGE-AUTH]] -- Storage auth uses JWT cache
- [[PROC-REALTIME-AUTH]] -- Realtime uses tenant cache for auth
- [[PROC-STUDIO-AUTH]] -- Studio uses React Query cache
- [[PROC-CLIENT-SDK-AUTH]] -- SDK uses localStorage for session cache
