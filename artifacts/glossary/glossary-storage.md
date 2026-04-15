---
id: "GLOSSARY-STORAGE"
type: "glossary"
title: "Storage Domain Glossary"
domain: "storage"
status: "draft"
last_verified: 2026-04-15
freshness_note: "Deep pass: traced connection.ts, knex.ts, storage.ts, object.ts, uploader.ts, locator.ts, limits.ts, renderer/, scanner/, cdn/, events/, backend/, config.ts, signature-v4.ts"
freshness_triggers:
  - "src/storage/"
  - "src/http/routes/"
  - "src/http/plugins/"
  - "src/internal/database/connection.ts"
  - "src/config.ts"
known_unknowns:
  - "Some terms may have business meanings not captured in code"
  - "Exact RLS policy names and definitions live in tenant databases, not in this codebase"
tags:
  - glossary
aliases: []
relates_to:
  - type: "parent"
    target: "[[SYS-STORAGE]]"
  - type: "refines"
    target: "[[GLOSSARY-SUPABASE-REEF]]"
  - type: "related"
    target: "[[SCH-STORAGE]]"
sources:
  - category: "implementation"
    type: "code"
    ref: "src/storage/storage.ts"
    notes: "Storage class — central orchestrator with from(), asSuperUser(), renderer() methods"
  - category: "implementation"
    type: "code"
    ref: "src/storage/object.ts"
    notes: "ObjectStorage — per-bucket object operations"
  - category: "implementation"
    type: "code"
    ref: "src/storage/uploader.ts"
    notes: "Uploader class — upload validation and streaming"
  - category: "implementation"
    type: "code"
    ref: "src/storage/locator.ts"
    notes: "StorageObjectLocator interface, TenantLocation, PassThroughLocation"
  - category: "implementation"
    type: "code"
    ref: "src/storage/backend/adapter.ts"
    notes: "StorageBackendAdapter abstract class"
  - category: "implementation"
    type: "code"
    ref: "src/storage/backend/index.ts"
    notes: "createStorageBackend factory — selects FileBackend or S3Backend"
  - category: "implementation"
    type: "code"
    ref: "src/storage/backend/file.ts"
    notes: "FileBackend — local filesystem backend using fs-xattr for metadata"
  - category: "implementation"
    type: "code"
    ref: "src/internal/database/connection.ts"
    notes: "TenantConnection — setScope(), asSuperUser(), pool management"
  - category: "implementation"
    type: "code"
    ref: "src/storage/database/knex.ts"
    notes: "StorageKnexDB — testPermission(), withTransaction()"
  - category: "implementation"
    type: "code"
    ref: "src/storage/renderer/renderer.ts"
    notes: "Renderer abstract class — render(), setHeaders(), handleCacheControl()"
  - category: "implementation"
    type: "code"
    ref: "src/storage/renderer/image.ts"
    notes: "ImageRenderer — imgproxy delegation with TransformOptions"
  - category: "implementation"
    type: "code"
    ref: "src/storage/scanner/scanner.ts"
    notes: "ObjectScanner — listOrphaned(), S3-vs-DB reconciliation"
  - category: "implementation"
    type: "code"
    ref: "src/storage/cdn/cdn-cache-manager.ts"
    notes: "CdnCacheManager — purge() via external CDN endpoint"
  - category: "implementation"
    type: "code"
    ref: "src/storage/limits.ts"
    notes: "Key validation, bucket name validation, file size parsing, empty folder detection"
  - category: "implementation"
    type: "code"
    ref: "src/storage/events/workers.ts"
    notes: "registerWorkers() — all pg-boss background job types"
  - category: "implementation"
    type: "code"
    ref: "src/http/plugins/signature-v4.ts"
    notes: "S3 Signature V4 authentication plugin"
  - category: "implementation"
    type: "code"
    ref: "src/config.ts"
    notes: "All configurable env vars including role names, backend type, feature flags"
notes: ""
---

## Terms

| Term | Definition | Code Location | Disambiguation / See Also |
|------|-----------|---------------|---------------------------|
| Bucket | A named container for objects with public/private visibility, optional file size limits, and allowed MIME type constraints. ID is user-defined text, not UUID. Three bucket types exist: STANDARD, ANALYTICS, VECTOR. | `src/http/routes/bucket/`, `src/storage/schemas/bucket.ts` | Not an S3 bucket -- Storage uses a single S3 bucket internally and multiplexes via tenant/bucket/object paths. See also BucketType. |
| BucketType | Postgres enum with three values: STANDARD (file storage), ANALYTICS (Iceberg), VECTOR (embedding storage). Determines which table family the bucket belongs to. | `src/storage/limits.ts`, `migrations/tenant/0038-iceberg-catalog-flag-on-buckets.sql` | Each type has its own bucket table: `buckets`, `buckets_analytics`, `buckets_vectors`. |
| Object | A file stored in Storage. Metadata (name, owner, version, MIME type) lives in Postgres `storage.objects` table; actual bytes live in the S3 backend. The `name` column stores the full path including folder segments. | `src/storage/object.ts`, `src/storage/schemas/object.ts` | Not to be confused with S3 objects in the backend -- Storage objects have a dual existence (metadata row + backend blob). |
| Backend | The blob storage provider where actual file bytes are stored. Abstracted behind `StorageBackendAdapter` with two implementations: `S3Backend` (AWS SDK v3, production default) and `FileBackend` (local filesystem, dev/test). Selected via `STORAGE_BACKEND` env var. | `src/storage/backend/adapter.ts`, `src/storage/backend/index.ts` | The `createStorageBackend()` factory in `index.ts` is the single creation point. |
| StorageBackendAdapter | Abstract class defining the blob storage contract: `getObject`, `uploadObject`, `deleteObject`, `copyObject`, `headObject`, `privateAssetUrl`, plus multipart methods. All methods throw "not implemented" by default. | `src/storage/backend/adapter.ts` | FileBackend extends this (implements all methods); S3Backend extends this separately. |
| FileBackend | Local filesystem backend for development. Stores files under a configurable root path, uses filesystem extended attributes (`fs-xattr`) for cache-control and content-type metadata. Supports mtime or md5 ETag algorithms. | `src/storage/backend/file.ts` | Platform-specific xattr keys: `com.apple.metadata.supabase.*` on macOS, `user.supabase.*` on Linux. |
| Storage | The central orchestrator class. Takes a backend, database, and locator. Provides `from(bucketId)` to get an `ObjectStorage` instance, `asSuperUser()` to bypass RLS, and `renderer(type)` to create renderers. | `src/storage/storage.ts` | This is the business logic layer, not the service name. All protocols instantiate operations through this class. |
| ObjectStorage | Per-bucket object operations class. Created via `Storage.from(bucketId)`. Handles upload, download, copy, move, delete, list, signed URLs. Contains an `Uploader` instance. | `src/storage/object.ts` | Always scoped to a single bucket. |
| TUS | Resumable upload protocol (tus.io). Enables large file uploads to be paused and resumed across unreliable connections. Storage implements the TUS server protocol. | `src/storage/protocols/tus/` | Distinct from S3 multipart uploads -- TUS is a separate protocol with its own endpoint (`/upload/resumable`). |
| Multipart Upload | S3-compatible chunked upload for large files. Tracked by two Postgres tables: `s3_multipart_uploads` (session) and `s3_multipart_uploads_parts` (individual parts with ETags). Parts cascade-delete when the upload is deleted. | `src/storage/protocols/s3/s3-handler.ts`, `src/storage/database/knex.ts` | Only used via the S3 protocol, not the HTTP REST API. |
| Signed URL | A time-limited URL granting access to a private object without requiring a JWT. Implemented as a JWT signed with the tenant's secret, containing the object path and expiry. | `src/http/routes/object/getSignedUploadURL.ts`, `src/storage/object.ts` | Two variants: signed download URLs and signed upload URLs. |
| Render | Image transformation applied on-the-fly during download. Four renderer types: `asset` (plain download), `head` (metadata only), `image` (imgproxy transformation), `info` (object info). | `src/http/routes/render/`, `src/storage/renderer/` | "Render" in Storage context always means serving an object, not just image transformation. |
| ImageRenderer | Renderer subclass that proxies requests to an external imgproxy service for on-the-fly image transformation (resize, crop, format conversion). Supports width, height, resize mode, format (origin/avif), and quality options. | `src/storage/renderer/image.ts` | Feature-flagged via `IMAGE_TRANSFORMATION_ENABLED`. Not available on all plans. |
| TransformOptions | Parameters for image transformation: `width`, `height`, `resize` (cover/contain/fill), `format` (origin/avif), `quality`. Width and height have configurable min/max limits. | `src/storage/renderer/image.ts:14-20` | Passed via query parameters on render routes. |
| Locator | Maps logical object paths (tenant/bucket/object) to physical storage keys in the backend. Interface `StorageObjectLocator` with two implementations: `TenantLocation` (prepends tenantId and bucketId) and `PassThroughLocation` (uses objectName as-is). | `src/storage/locator.ts` | TenantLocation is the default in multitenant mode. PassThroughLocation used for direct backend access. |
| Scanner | `ObjectScanner` class that reconciles objects between Postgres metadata and the S3 backend. Uses async generators to yield orphaned keys (objects in S3 but not in DB, or vice versa). Creates a temporary table to hold S3 keys during comparison. | `src/storage/scanner/scanner.ts` | Used by admin operations for data integrity checks, not by normal request flow. |
| Iceberg | Apache Iceberg REST Catalog protocol for table-format data lake access. Uses separate tables (`buckets_analytics`, `iceberg_namespaces`, `iceberg_tables`) from standard storage. Supports sigv4 and token auth. | `src/http/routes/iceberg/` | Buckets of type ANALYTICS. Completely separate data model from STANDARD buckets. |
| Vector | Experimental vector/embedding storage protocol. Uses `buckets_vectors` and `vector_indexes` tables with dimension and distance metric configuration. Routes restricted to `service_role` JWT only. | `src/http/routes/vector/`, `src/storage/schemas/vector.ts` | Buckets of type VECTOR. Separate from both STANDARD and ANALYTICS buckets. |
| Uploader | Core upload handler class that validates file size limits, MIME types, custom metadata size (max 1MB), and streams bytes to the backend. Uses `canUpload()` with `testPermission()` to pre-check RLS before streaming. | `src/storage/uploader.ts` | Three upload types tracked: `standard`, `s3`, `resumable`. |
| TenantConnection | Database connection wrapper that holds pool reference, user credentials (JWT payload + role), and the `setScope()` method that injects 10 GUC variables into each Postgres transaction for RLS evaluation. | `src/internal/database/connection.ts` | Not a raw DB connection -- it wraps a Knex pool strategy and manages role switching. |
| setScope | Method on `TenantConnection` that executes `SELECT set_config(...)` to inject role, JWT claims, request headers, path, method, and operation into the current Postgres transaction. This is what enables RLS policies to reference `current_setting('request.jwt.claims')`. | `src/internal/database/connection.ts:170-197` | Called at the start of every transaction via `withTransaction()`. All `set_config` calls are transaction-local (3rd arg = `true`). |
| asSuperUser | Method available on both `TenantConnection` and `Storage`/`StorageKnexDB` that returns a new instance using admin (service_role) credentials, bypassing RLS for internal operations like background deletes and bucket management. | `src/internal/database/connection.ts:157-168`, `src/storage/storage.ts:50-52`, `src/storage/database/knex.ts:97-103` | Not a Postgres superuser -- it uses the `service_role` database role which typically has RLS bypass. |
| testPermission | Method on `StorageKnexDB` that runs a callback inside a transaction that always rolls back via `TestPermissionRollbackError`. Used to check if RLS would allow an operation without persisting any data. | `src/storage/database/knex.ts:105-118` | Used by `Uploader.canUpload()` to avoid orphaned S3 blobs when permission is denied. |
| CdnCacheManager | Manages CDN cache invalidation by calling an external purge endpoint (configured via `CDN_PURGE_ENDPOINT_URL`). Sends tenant, bucket, and object name to purge stale cached content. | `src/storage/cdn/cdn-cache-manager.ts` | Only active when CDN purge endpoint is configured. |
| Prefix | Auto-materialized folder path entry in the `storage.prefixes` table. Created by BEFORE INSERT triggers on `objects` and cleaned up by AFTER DELETE triggers. Composite PK of `(bucket_id, level, name)` where level counts path segments. | `migrations/tenant/0026-objects-prefixes.sql` | Not created explicitly by users -- entirely trigger-maintained. Enables efficient folder listing. |
| Version | UUID string appended to object keys in the backend (using `FILE_VERSION_SEPARATOR` or `/`). Allows object overwrites without clobbering the backend blob until the old version is garbage-collected. | `src/storage/backend/adapter.ts:247-249`, `src/storage/locator.ts` | Not S3 versioning -- Storage implements its own versioning scheme separate from S3 bucket versioning. |
| SignatureV4 | AWS Signature Version 4 authentication used by the S3-compatible protocol. Verifies request signatures using access key/secret pairs. Supports both header-based and query-string-based (presigned URL) authentication. | `src/http/plugins/signature-v4.ts`, `src/storage/protocols/s3/` | In multitenant mode, S3 credentials are managed per-tenant via the admin API. |
| pg-boss | Postgres-based job queue library used for background processing. Workers handle: object deletion, webhooks, migrations, JWKS key rotation, Iceberg catalog reconciliation, and PgBoss self-upgrades. | `src/storage/events/workers.ts`, `src/start/worker.ts` | 13 registered worker types. Runs in a separate process entry point (`worker.ts`). |
| Empty Folder Placeholder | A zero-byte object with name ending in `.emptyFolderPlaceholder`. Used to represent empty folders since object storage is flat by nature. | `src/storage/limits.ts:137-139` | Detected by `isEmptyFolder()` function. Convention inherited from Supabase client SDKs. |
| Shard | A capacity-limited allocation unit for vector or iceberg-table resources in multitenant mode. Managed by `ShardCatalog` backed by the multitenant database. | `src/start/server.ts` | Only relevant in multitenant deployments. Not related to database sharding. |

## Naming Conventions

- Route modules organized by protocol/resource: `object/`, `bucket/`, `s3/`, `tus/`, `iceberg/`, `vector/`, `render/`, `cdn/`
- Internal modules use `@internal/` path alias (e.g., `@internal/auth`, `@internal/errors`, `@internal/monitoring`, `@internal/database`)
- Storage modules use `@storage/` path alias (e.g., `@storage/backend`, `@storage/database`, `@storage/locator`, `@storage/schemas`)
- Storage schema tables: `storage.buckets`, `storage.objects`, `storage.prefixes`, `storage.s3_multipart_uploads`, `storage.s3_multipart_uploads_parts`, `storage.buckets_analytics`, `storage.iceberg_namespaces`, `storage.iceberg_tables`, `storage.buckets_vectors`, `storage.vector_indexes`
- Three configurable database roles: `anon` (unauthenticated), `authenticated` (valid JWT), `service_role` (admin/internal)
- GUC variable namespace: `request.*` for HTTP context, `storage.*` for service-specific settings

## Related

- [[SYS-STORAGE]] -- parent system
- [[SCH-STORAGE]] -- full data model with table definitions
- [[GLOSSARY-SUPABASE-REEF]] -- unified cross-service glossary
- [[DEC-STORAGE-RLS-AUTH-DELEGATION]] -- RLS auth delegation decision
- [[DEC-MULTI-PROTOCOL-STORAGE]] -- multi-protocol architecture decision
