---
id: "SYS-STORAGE"
type: "system"
title: "Supabase Storage Service"
domain: "storage"
status: "draft"
last_verified: 2026-04-15
freshness_note: "Updated with deep read of config, docker-compose, server/worker entry points, and protocol directories"
freshness_triggers:
  - "docker-compose.yml"
  - "docker-compose-multi-tenant.yml"
  - "package.json"
  - "src/admin-app.ts"
  - "src/app.ts"
  - "src/config.ts"
  - "src/http/routes/index.ts"
  - "src/start/server.ts"
  - "src/start/worker.ts"
  - "src/storage/storage.ts"
  - "watt.json"
known_unknowns:
  - "CDN purge endpoint integration details — config has CDN_PURGE_ENDPOINT_URL/KEY but purge logic not traced"
  - "Webhook consumer contract — config supports WEBHOOK_URL but downstream consumer not documented"
  - "Kubernetes cluster discovery mechanism used by @internal/cluster"
  - "Supavisor connection pooler configuration details in multitenant mode"
tags:
  - fastify
  - iceberg
  - multiprotocol
  - multitenant
  - rls
  - s3
  - storage
  - tus
  - typescript
aliases: []
relates_to:
  - type: "refines"
    target: "[[API-STORAGE]]"
  - type: "refines"
    target: "[[CON-AUTH-STORAGE]]"
  - type: "refines"
    target: "[[DEC-MULTI-PROTOCOL-STORAGE]]"
  - type: "refines"
    target: "[[GLOSSARY-STORAGE]]"
  - type: "refines"
    target: "[[PROC-STORAGE-AUTH]]"
  - type: "refines"
    target: "[[RISK-STORAGE-KNOWN-GAPS]]"
  - type: "refines"
    target: "[[SCH-STORAGE]]"
  - type: "depends_on"
    target: "[[SYS-AUTH]]"
  - type: "feeds"
    target: "[[SYS-CLIENT-SDK]]"
sources:
  - category: "implementation"
    type: "github"
    ref: "docker-compose-multi-tenant.yml"
  - category: "implementation"
    type: "github"
    ref: "docker-compose.yml"
  - category: "implementation"
    type: "github"
    ref: "package.json"
  - category: "implementation"
    type: "github"
    ref: "src/admin-app.ts"
  - category: "implementation"
    type: "github"
    ref: "src/app.ts"
  - category: "implementation"
    type: "github"
    ref: "src/config.ts"
  - category: "implementation"
    type: "github"
    ref: "src/http/routes/index.ts"
  - category: "implementation"
    type: "github"
    ref: "src/start/server.ts"
  - category: "implementation"
    type: "github"
    ref: "src/start/worker.ts"
  - category: "implementation"
    type: "github"
    ref: "src/storage/backend/adapter.ts"
  - category: "implementation"
    type: "github"
    ref: "src/storage/storage.ts"
  - category: "implementation"
    type: "github"
    ref: "watt.json"
  - category: "context"
    type: "doc"
    ref: "sources/context/decisions/storage-overview.md"
    notes: "Official Supabase documentation describing three bucket types (Files, Analytics, Vector)"
notes: ""
---

## Overview

Storage is a TypeScript object storage service providing multi-protocol access to file storage. It supports four primary protocols: HTTP/REST (standard CRUD), TUS (resumable uploads), S3-compatible API, and Iceberg REST Catalog, plus experimental Vector storage. Postgres stores all metadata (buckets, objects, multipart uploads) while actual file bytes live in an S3-compatible backend. Authorization uses Postgres Row Level Security policies evaluated at request time using the caller's JWT. The service runs in both single-tenant and multitenant modes, with the latter adding an admin API server, a separate multitenant database, PgBouncer/Supavisor connection pooling, and shard-based resource allocation.

## Key Facts

- TypeScript/Fastify v5 HTTP server (package `supa-storage` v1.11.2) requiring Node >= 24 → `package.json`
- Two Fastify apps: main app on port 5000 (`src/app.ts`) and admin app on port 5001 (`src/admin-app.ts`); admin app only starts in multitenant mode → `src/start/server.ts`
- Nine route groups registered on main app: object, bucket, tus, s3, render, cdn, healthcheck, iceberg, vector → `src/app.ts`
- Admin app exposes tenant management, object admin, JWKS config, migration, S3 credential, queue, and metrics-config routes → `src/admin-app.ts`
- Storage backend abstraction supports two types: `file` and `s3`, selected via `STORAGE_BACKEND` env var → `src/config.ts`
- JWT auth uses `jose` library with support for HMAC (HS256/384/512), RSA, EC, and EdDSA algorithms, plus JWKS key rotation → `src/internal/auth/jwt.ts`
- Background job processing via pg-boss (Postgres-based queue) with separate server and worker entry points → `src/start/server.ts`, `src/start/worker.ts`
- Multitenant mode uses a separate multitenant database, Supavisor connection pooler, and progressive/on-request/full-fleet migration strategies → `src/config.ts`, `docker-compose-multi-tenant.yml`
- Platformatic/Watt runtime wraps the Node process with configurable worker count, health checks, and management API → `watt.json`
- Image transformation delegated to external imgproxy service, toggled via `IMAGE_TRANSFORMATION_ENABLED` → `src/config.ts`, `docker-compose.yml`
- Rate limiting supports memory and Redis drivers, configurable per render-path max requests/sec → `src/config.ts`
- Iceberg REST Catalog protocol supports sigv4 and token auth types with configurable catalog, namespace, and table count limits → `src/config.ts`
- Vector storage protocol uses dedicated S3 buckets with configurable max buckets and indexes counts → `src/config.ts`
- ShardCatalog allocates capacity for vector and iceberg-table shards at startup in multitenant mode → `src/start/server.ts`
- Cluster module monitors cluster size changes and triggers connection pool rebalancing across workers → `src/start/server.ts`

## Responsibilities

- **Owns**: Object storage metadata, bucket management, file upload/download, signed URLs, image rendering/transformation, S3 protocol compatibility, TUS resumable uploads, Iceberg REST Catalog proxy, Vector storage API, background queue processing (delete, webhooks)
- **Does not own**: User authentication (delegates to Auth JWTs), RLS policy definitions (lives in user databases), actual blob storage (delegates to S3 backend), image transformation processing (delegates to imgproxy), connection pooling in multitenant mode (delegates to Supavisor/PgBouncer)

## Dependencies

| System | Integration Type | Purpose | Auth Method |
|--------|-----------------|---------|-------------|
| PostgreSQL (tenant DB) | Direct connection via Knex | Object/bucket metadata, RLS policy evaluation | Database roles (anon, authenticated, service_role) |
| PostgreSQL (multitenant DB) | Direct connection via Knex | Tenant registry, shard catalog, migration tracking | Database credentials |
| S3-compatible backend (MinIO/AWS) | AWS SDK v3 | Blob storage for file bytes | AWS access key/secret |
| imgproxy | HTTP proxy | Image transformation and rendering | Internal network (no auth) |
| pg-boss (Postgres queue) | Embedded library | Background job processing (deletes, webhooks) | Shared DB connection |
| Supavisor / PgBouncer | Connection pooler | Database connection pooling in multitenant mode | Database credentials |
| Redis (optional) | ioredis client | Rate limiting driver | Redis URL |
| Iceberg REST Catalog | HTTP client | Table metadata for Iceberg protocol | sigv4 or bearer token |
| OpenTelemetry collector | OTLP gRPC | Metrics and traces export | Internal network |
| CDN purge endpoint | HTTP client | Cache invalidation | API key header |

## Does NOT Own

- **User authentication**: Storage validates JWTs but never issues them. Auth service ([[SYS-AUTH]]) owns identity.
- **RLS policies**: Row Level Security rules live in the tenant's Postgres database, defined by the project owner. Storage only evaluates them via `SET ROLE` at query time.
- **Blob byte storage**: Actual file content lives in an external S3-compatible backend. Storage manages metadata and proxies reads/writes.
- **Image transformation logic**: Transformation is delegated to an external imgproxy instance. Storage routes requests and caches results.
- **Connection pooling**: In multitenant mode, Supavisor or PgBouncer owns connection multiplexing. Storage connects through the pool URL.
- **Webhook consumers**: Storage enqueues webhook events via pg-boss but does not own the downstream consumer.

## Storage Bucket Types

Storage organizes resources into three specialized bucket types, each optimized for a different use case → `sources/context/decisions/storage-overview.md`:

- **Files buckets** (STANDARD) — Traditional file storage for images, videos, documents, and general-purpose content. Features global CDN delivery, image optimization/transformation, RLS integration, and direct URL access.
- **Analytics buckets** (ANALYTICS) — Purpose-built for data lakes using Apache Iceberg table format. Supports SQL access via Postgres foreign tables, partitioned data organization, and efficient analytical querying. Backed by the `buckets_analytics`, `iceberg_namespaces`, and `iceberg_tables` entities.
- **Vector buckets** (VECTOR) — Specialized storage for vector embeddings and AI/ML similarity search. Supports HNSW and Flat index types with cosine, euclidean, and L2 distance metrics. Backed by the `buckets_vectors` and `vector_indexes` entities.

## Domain Behavior Highlights

- **Multi-protocol single engine**: All four protocols (HTTP, TUS, S3, Iceberg) plus Vector share the same `Storage` class (`src/storage/storage.ts`) as their business logic layer, ensuring consistent authorization and metadata handling regardless of access method.
- **Protocol-specific route isolation**: Each protocol is a separate Fastify plugin registered at a distinct prefix (`/object`, `/upload/resumable`, `/s3`, `/iceberg`, `/vector`), enabling independent middleware stacks and versioning per protocol → `src/app.ts`.
- **Dual-mode tenant architecture**: A single codebase runs in both single-tenant (one DB, no admin app) and multitenant (tenant registry DB + per-tenant DBs, admin app on separate port, shard catalog) modes, controlled by `MULTI_TENANT` env var → `src/config.ts`, `src/start/server.ts`.
- **Graceful shutdown orchestration**: Uses `AsyncAbortController` with chained signal groups to shut down HTTP servers, queue workers, PubSub listeners, and connection pools in deterministic order → `src/start/server.ts`, `src/start/worker.ts`.
- **Shard-aware resource allocation**: In multitenant mode, vector and iceberg resources are pre-allocated as shards with configurable capacity, managed by `ShardCatalog` backed by the multitenant Knex store → `src/start/server.ts`.

## Runtime Components

| Component | Tech Stack | Purpose | Entry Point |
|-----------|-----------|---------|-------------|
| API Server | Fastify 5 on Node >= 24 | Main HTTP server handling all storage protocols | `src/start/server.ts` → `src/app.ts` |
| Admin Server | Fastify 5 | Tenant management, migrations, S3 credentials (multitenant only) | `src/start/server.ts` → `src/admin-app.ts` |
| Queue Worker | pg-boss on PostgreSQL | Background job processing (deletes, webhooks) | `src/start/worker.ts` |
| Platformatic/Watt Runtime | @platformatic/node | Multi-worker process management, health checks | `watt.json` |
| Storage Engine | Custom TypeScript classes | Core business logic: Storage, ObjectStorage, Uploader | `src/storage/storage.ts` |
| S3 Backend Adapter | @aws-sdk/client-s3 | Blob read/write/delete against S3-compatible stores | `src/storage/backend/s3/` |
| Image Renderer | HTTP proxy to imgproxy | Image transformation pipeline | `src/storage/renderer/` |
| PubSub | pg-listen | Real-time tenant config updates via PostgreSQL NOTIFY | `src/internal/pubsub/` |
| Cluster Monitor | @kubernetes/client-node | Cluster size tracking, connection pool rebalancing | `src/internal/cluster/` |

## Core Concepts

- **Bucket**: A named container for objects. Buckets have public/private visibility settings and file size/type constraints.
- **Object**: A file stored in a bucket. Metadata in Postgres, bytes in S3 backend.
- **Protocol**: One of the access methods (HTTP, TUS, S3, Iceberg, Vector) sharing the same storage engine.
- **Storage Engine**: Core business logic layer (`Storage` class) that mediates between protocols, database, and backend.
- **Backend**: The actual file storage provider (S3-compatible or local filesystem). Abstracted via `StorageBackendAdapter`.
- **Uploader**: Handles file upload with validation (size limits, MIME types) and byte-level streaming.
- **Shard**: A capacity-limited allocation unit for vector or iceberg-table resources in multitenant mode.
- **Tenant**: An isolated project with its own Postgres database, connection pool, and RLS policies. Identified by `PROJECT_REF` or `TENANT_ID`.

## Related

- [[SCH-STORAGE]] -- data model details
- [[API-STORAGE]] -- API endpoint reference
- [[PROC-STORAGE-AUTH]] -- authentication mechanism
- [[GLOSSARY-STORAGE]] -- domain terminology
- [[RISK-STORAGE-KNOWN-GAPS]] -- known risks
- [[CON-AUTH-STORAGE]] -- auth integration contract
- [[DEC-MULTI-PROTOCOL-STORAGE]] -- multi-protocol architecture decision
- [[SYS-AUTH]] -- provides JWTs for RLS authorization
- [[SYS-CLIENT-SDK]] -- storage-js sub-package wraps Storage API
