---
id: "PAT-CROSS-SERVICE-STORAGE"
type: "pattern"
title: "Cross-Service Storage and File Handling Pattern"
domain: "supabase-reef"
status: draft
last_verified: 2026-04-15
freshness_note: "scuba-depth cross-service pattern analysis"
freshness_triggers:
  - "src/storage/object.ts"
  - "src/storage/uploader.ts"
  - "src/http/plugins/storage.ts"
  - "src/http/plugins/signature-v4.ts"
  - "packages/core/storage-js/src/"
  - "apps/studio/data/storage/"
  - "apps/studio/lib/upload.ts"
  - "internal/mailer/templatemailer/template.go"
known_unknowns:
  - "Full list of Storage backend adapters beyond S3"
  - "Whether Auth stores any files directly (avatars, email templates) or delegates to Storage"
  - "Studio's storage browser internal pagination strategy for large buckets"
tags:
  - pattern
  - cross-service
  - storage
  - files
aliases:
  - "File Handling Pattern"
relates_to:
  - type: "synthesizes"
    target: "[[PROC-STORAGE-AUTH]]"
  - type: "synthesizes"
    target: "[[PROC-STUDIO-AUTH]]"
  - type: "synthesizes"
    target: "[[PROC-CLIENT-SDK-AUTH]]"
  - type: "related"
    target: "[[SYS-STORAGE]]"
  - type: "related"
    target: "[[PAT-CROSS-SERVICE-AUTH]]"
  - type: "related"
    target: "[[PAT-CROSS-SERVICE-QUEUE]]"
sources:
  - category: "implementation"
    type: "github"
    ref: "src/storage/object.ts"
  - category: "implementation"
    type: "github"
    ref: "src/http/plugins/db.ts"
  - category: "implementation"
    type: "github"
    ref: "src/http/plugins/signature-v4.ts"
  - category: "implementation"
    type: "github"
    ref: "apps/studio/data/storage/buckets-query.ts"
  - category: "implementation"
    type: "github"
    ref: "apps/studio/lib/upload.ts"
  - category: "implementation"
    type: "github"
    ref: "packages/core/supabase-js/src/SupabaseClient.ts"
  - category: "implementation"
    type: "github"
    ref: "src/storage/events/lifecycle/object-created.ts"
notes: ""
---

## Overview

File/object storage in Supabase flows through the Storage service as the central hub, but is consumed and managed from Studio (dashboard UI), the Client SDK (developer API), and indirectly by Auth (which references stored assets). The Storage service itself provides three protocol frontends (REST, S3, TUS) while Studio and the Client SDK interact via the REST/HTTP API. The pattern shows a clear layered architecture: Client SDK and Studio are thin API clients, Storage is the authoritative backend with RLS-enforced database state and pluggable object backends.

## Key Facts

- Storage's `ObjectStorage` class orchestrates all object operations (upload, copy, move, delete) using a `StorageBackendAdapter` for physical storage and a `Database` for metadata, with RLS enforced via `TenantConnection.setScope` → `src/storage/object.ts`
- Storage supports `asSuperUser()` which creates a new ObjectStorage instance that bypasses RLS, used for admin operations → `src/storage/object.ts`
- Storage emits lifecycle events for all object mutations: `ObjectCreated:Put`, `ObjectCreated:Post`, `ObjectCreated:Copy`, `ObjectCreated:Move`, `ObjectRemoved`, `ObjectUpdatedMetadata` → `src/storage/events/lifecycle/object-created.ts`
- Storage events are sent to a PgBoss queue and include metadata like `uploadType` (standard/resumable/s3), `bucketId`, and object `version` → `src/storage/events/lifecycle/object-created.ts`
- The S3 protocol path in Storage (Signature V4) converges with the REST path by synthesizing a JWT from S3 credentials, so downstream object operations are protocol-agnostic → `src/http/plugins/signature-v4.ts`
- Studio accesses storage buckets via typed openapi-fetch calls to `/platform/storage/{ref}/buckets` endpoints, using the same auth middleware pattern as all Studio API calls → `apps/studio/data/storage/buckets-query.ts`
- Studio has a dedicated `uploadAttachment` utility that creates a temporary Supabase client (with `persistSession: false`) to upload support attachments to a separate Supabase project → `apps/studio/lib/upload.ts`
- Client SDK's `StorageClient` is initialized with the `fetchWithAuth` wrapper, inheriting the same auth token injection as all other sub-clients → `packages/core/supabase-js/src/SupabaseClient.ts`
- Storage's DB plugin creates a Postgres connection per request with both user JWT payload and a service-key super-user, disposing connections on `onSend`, `onTimeout`, and `onRequestAbort` hooks → `src/http/plugins/db.ts`
- Storage's `dbSuperUser` plugin bypasses per-user JWT claim scoping for admin operations, effectively disabling RLS → `src/http/plugins/db.ts`
- Auth does not directly use the Storage service for file handling; email templates are rendered from Go template strings, not stored files → `internal/mailer/templatemailer/template.go`

## Where It Appears

| Aspect | Storage (backend) | Studio (dashboard) | Client SDK | Auth |
|--------|-------------------|-------------------|------------|------|
| **Role** | Authoritative object store with RLS | Management UI for buckets/objects | Developer-facing upload/download API | Does not use Storage directly |
| **Protocol** | REST, S3 (Sig V4), TUS | REST via platform API proxy | REST via storage-js | N/A |
| **Auth integration** | JWT claims injected into Postgres session for RLS | Bearer token from GoTrue session | `fetchWithAuth` injects token | N/A |
| **Upload mechanism** | Fastify routes + Uploader class | openapi-fetch POST to platform API | `supabase.storage.from(bucket).upload()` | N/A |
| **Event emission** | PgBoss queue lifecycle events | N/A (consumer only) | N/A (consumer only) | N/A |
| **Backend adapter** | Pluggable (S3, file system) | N/A | N/A | N/A |

## Design Intent

The layered design ensures that all access control is centralized in the Storage service via Postgres RLS, regardless of whether the request arrives from the Client SDK, Studio, or the S3 protocol. Studio and the Client SDK are thin API consumers that do not implement storage logic themselves. Lifecycle events (object created/removed) enable async processing (webhooks, vector indexing) without blocking the upload path.

## Trade-offs

- **Three protocol frontends**: REST, S3, and TUS increase API surface and maintenance burden, but allow compatibility with existing S3 tooling and resumable uploads for large files.
- **RLS on every request**: Every storage operation runs within an RLS-scoped Postgres transaction, adding per-request overhead but ensuring consistent authorization.
- **Studio uploads via separate client**: Studio's `uploadAttachment` creates a throwaway Supabase client for a different project, which is a pragmatic workaround but could confuse developers reading the code.

## Agent Guidance

- When debugging storage access issues, check both the JWT role and the RLS policies on `storage.buckets` and `storage.objects` tables. The `set_config` variables injected by `TenantConnection.setScope` are what RLS policies evaluate.
- The Storage service is the only place where physical files are written to backends. Studio and Client SDK only interact via HTTP APIs.
- S3 protocol requests go through a completely different auth path (Signature V4) but converge to the same `request.jwt` / `request.jwtPayload` interface before reaching object handlers.
- Auth service does not store or retrieve files from Storage. Any file references in Auth (like avatars) are URLs, not direct Storage API calls.

## Related

- [[SYS-STORAGE]] -- Storage system overview
- [[PROC-STORAGE-AUTH]] -- Storage authentication details
- [[PROC-STUDIO-AUTH]] -- Studio storage access
- [[PROC-CLIENT-SDK-AUTH]] -- Client SDK storage access
- [[PAT-CROSS-SERVICE-AUTH]] -- Auth pattern comparison
- [[PAT-CROSS-SERVICE-QUEUE]] -- Queue pattern used for Storage lifecycle events
