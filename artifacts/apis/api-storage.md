---
id: "API-STORAGE"
type: "api"
title: "Storage Multi-Protocol API"
domain: "storage"
status: active
last_verified: 2026-04-15
freshness_note: "Deep pass against OpenAPI spec, route source files, S3 router, TUS lifecycle, and operations registry"
freshness_triggers:
  - "src/http/routes/bucket/"
  - "src/http/routes/index.ts"
  - "src/http/routes/object/"
  - "src/http/routes/operations.ts"
  - "src/http/routes/s3/"
  - "src/http/routes/tus/"
known_unknowns:
  - "Rate limiting configuration per protocol"
  - "Admin app endpoints and their separation from main app"
tags:
  - storage
  - rest-api
  - s3
  - tus
  - iceberg
  - vector
aliases: []
relates_to:
  - type: "depends_on"
    target: "[[SCH-STORAGE]]"
  - type: "parent"
    target: "[[SYS-STORAGE]]"
sources:
  - category: "implementation"
    type: "github"
    ref: "sources/apis/storage/openapi.json"
    notes: "Tier 4 extracted OpenAPI spec covering all protocol endpoints"
  - category: "implementation"
    type: "github"
    ref: "src/http/routes/bucket/index.ts"
    notes: "Bucket route registration with JWT + DB plugins"
  - category: "implementation"
    type: "github"
    ref: "src/http/routes/index.ts"
    notes: "Top-level route module exports"
  - category: "implementation"
    type: "github"
    ref: "src/http/routes/object/createObject.ts"
    notes: "Object upload route with multipart and upsert support"
  - category: "implementation"
    type: "github"
    ref: "src/http/routes/object/getSignedURL.ts"
    notes: "Signed URL generation with optional image transformation"
  - category: "implementation"
    type: "github"
    ref: "src/http/routes/object/index.ts"
    notes: "Object route split: authenticated vs public/signed scopes"
  - category: "implementation"
    type: "github"
    ref: "src/http/routes/operations.ts"
    notes: "Canonical operation name registry for all protocols"
  - category: "implementation"
    type: "github"
    ref: "src/http/routes/s3/router.ts"
    notes: "S3 command router with AJV validation and querystring-based dispatch"
  - category: "implementation"
    type: "github"
    ref: "src/http/routes/tus/lifecycle.ts"
    notes: "TUS lifecycle hooks: auth, naming, upload completion"
notes: ""
---

## Overview

Storage exposes four parallel protocol interfaces through a Fastify HTTP server. The primary HTTP/REST API handles bucket and object CRUD with JWT-based auth. An S3-compatible protocol replicates core AWS S3 operations via a custom querystring+header router. TUS resumable uploads provide reliable large-file transfers. Beyond file storage, the service also hosts an Apache Iceberg REST Catalog for analytics tables and a Vector storage API for similarity search. All protocols share the same underlying storage backend and database layer ([[SCH-STORAGE]]).

## Key Facts

- Ten route modules exported from the top-level index: admin, bucket, cdn, health, iceberg, object, render, s3, tus, vector -> `src/http/routes/index.ts`
- Object routes split into two Fastify scopes: authenticated (JWT + db + storage plugins) and public/signed (dbSuperUser + storage, no JWT) -> `src/http/routes/object/index.ts`
- Bucket routes register jwt, db, and storage plugins at scope level -- all bucket operations require Bearer JWT -> `src/http/routes/bucket/index.ts`
- Upload endpoint accepts multipart form data (max 10 fields, 1 file) and supports upsert via `x-upsert: true` header -> `src/http/routes/object/createObject.ts`
- Signed URL generation accepts an `expiresIn` (seconds) body param and optional `transform` object for image transformations -> `src/http/routes/object/getSignedURL.ts`
- S3 router dispatches 18 commands (including UploadPartCopy, ListMultipartUploads, ListParts) via querystring and header matching rather than path-only routing -> `src/http/routes/s3/router.ts`
- S3 schema validation uses AJV with `coerceTypes: 'array'`, `removeAdditional: true`, and fast-uri resolver -> `src/http/routes/s3/router.ts`
- TUS upload IDs encode tenant/bucket/object/version as base64url in the URL; `getFileIdFromRequest` decodes them back -> `src/http/routes/tus/lifecycle.ts`
- TUS signed uploads authenticate via `x-signature` header verified by `verifyObjectSignature`, bypassing JWT -> `src/http/routes/tus/lifecycle.ts`
- Operations registry defines 70+ canonical operation names (e.g. `storage.object.upload`, `storage.s3.object.get`, `storage.tus.upload.create`) used for tracing and access control -> `src/http/routes/operations.ts`
- Public object endpoints (`/object/public/`, `/render/image/public/`) require no authentication at all -> `sources/apis/storage/openapi.json`
- Image transformation rendering is available in three auth modes: authenticated, public, and signed URL -> `sources/apis/storage/openapi.json`

## Source of Truth

Route modules in `src/http/routes/`. Protocol handlers in `src/storage/protocols/`. Canonical operation names in `src/http/routes/operations.ts`. OpenAPI spec (tier 4, extracted from source) in `sources/apis/storage/openapi.json`.

## Resource Map

### HTTP/REST API -- Objects

| Method | Path | Operation | Auth |
|--------|------|-----------|------|
| POST | `/object/:bucketName/*` | Upload object | Bearer JWT |
| PUT | `/object/:bucketName/*` | Update (overwrite) object | Bearer JWT |
| GET | `/object/:bucketName/*` | Download object | Bearer JWT |
| DELETE | `/object/:bucketName/*` | Delete object | Bearer JWT |
| HEAD | `/object/:bucketName/*` | Get object info | Bearer JWT |
| DELETE | `/object/:bucketName` | Delete multiple objects | Bearer JWT |
| POST | `/object/list/:bucketName` | List objects (v1) | Bearer JWT |
| POST | `/object/ls/:bucketName` | List objects (v2) | Bearer JWT |
| POST | `/object/move` | Move object | Bearer JWT |
| POST | `/object/copy` | Copy object | Bearer JWT |
| POST | `/object/sign/:bucketName/*` | Create signed URL | Bearer JWT |
| POST | `/object/sign` | Create signed URLs (batch) | Bearer JWT |
| POST | `/object/upload/sign/:bucketName/*` | Create signed upload URL | Bearer JWT |
| GET | `/object/public/:bucketName/*` | Get public object | None |
| GET | `/object/sign/:token/:bucketName/*` | Get via signed URL | Token |
| PUT | `/object/sign/:token/:bucketName/*` | Upload via signed URL | Token |

### HTTP/REST API -- Buckets

| Method | Path | Operation | Auth |
|--------|------|-----------|------|
| POST | `/bucket` | Create bucket | Bearer JWT |
| GET | `/bucket` | List buckets | Bearer JWT |
| GET | `/bucket/:id` | Get bucket | Bearer JWT |
| PUT | `/bucket/:id` | Update bucket | Bearer JWT |
| DELETE | `/bucket/:id` | Delete bucket | Bearer JWT |
| POST | `/bucket/:id/empty` | Empty bucket | Bearer JWT |

### S3-Compatible Protocol

18 commands dispatched via querystring/header matching on `/s3`, `/s3/:Bucket`, `/s3/:Bucket/:Key`:

| Command | HTTP | Dispatch Key |
|---------|------|-------------|
| ListBuckets | GET `/s3` | default |
| CreateBucket | PUT `/s3/:Bucket` | default |
| DeleteBucket | DELETE `/s3/:Bucket` | default |
| HeadBucket | HEAD `/s3/:Bucket` | default |
| ListObjects | GET `/s3/:Bucket` | `?list-type` |
| GetObject | GET `/s3/:Bucket/:Key` | default |
| PutObject | PUT `/s3/:Bucket/:Key` | default |
| CopyObject | PUT `/s3/:Bucket/:Key` | `x-amz-copy-source` header |
| DeleteObject | DELETE `/s3/:Bucket/:Key` | default |
| HeadObject | HEAD `/s3/:Bucket/:Key` | default |
| CreateMultipartUpload | POST `/s3/:Bucket/:Key?uploads` | `?uploads` |
| UploadPart | PUT `/s3/:Bucket/:Key?uploadId` | `?uploadId` |
| UploadPartCopy | PUT `/s3/:Bucket/:Key` | `?uploadId` + `x-amz-copy-source` |
| CompleteMultipartUpload | POST `/s3/:Bucket/:Key?uploadId` | `?uploadId` |
| AbortMultipartUpload | DELETE `/s3/:Bucket/:Key?uploadId` | `?uploadId` |
| ListMultipartUploads | GET `/s3/:Bucket?uploads` | `?uploads` |
| ListParts | GET `/s3/:Bucket/:Key?uploadId` | `?uploadId` |
| DeleteObjects | POST `/s3/:Bucket?delete` | `?delete` |

### TUS Resumable Uploads

| Method | Path | Operation |
|--------|------|-----------|
| POST | `/upload/resumable` | Create upload |
| HEAD | `/upload/resumable/:id` | Get upload offset |
| PATCH | `/upload/resumable/:id` | Resume (send bytes) |
| DELETE | `/upload/resumable/:id` | Cancel upload |

### Image Transformation

| Method | Path | Auth |
|--------|------|------|
| GET | `/render/image/authenticated/:bucket/*` | Bearer JWT |
| GET | `/render/image/public/:bucket/*` | None |
| GET | `/render/image/sign/:token/:bucket/*` | Token |

### Iceberg REST Catalog

| Method | Path | Operation |
|--------|------|-----------|
| GET | `/iceberg/v1/config` | Get catalog config |
| GET/POST | `/iceberg/v1/namespaces` | List / Create namespace |
| GET/DELETE | `/iceberg/v1/namespaces/:ns` | Get / Delete namespace |
| GET/POST | `/iceberg/v1/namespaces/:ns/tables` | List / Create table |
| GET/DELETE | `/iceberg/v1/namespaces/:ns/tables/:tbl` | Get / Delete table |

### Vector Storage

| Method | Path | Operation |
|--------|------|-----------|
| GET/POST | `/vector/buckets` | List / Create vector bucket |
| GET/DELETE | `/vector/buckets/:id` | Get / Delete vector bucket |
| GET/POST | `/vector/buckets/:id/indexes` | List / Create index |
| GET/DELETE | `/vector/buckets/:id/indexes/:idx` | Get / Delete index |
| GET/PUT/DELETE | `/vector/buckets/:id/indexes/:idx/vectors` | List / Upsert / Delete vectors |
| POST | `/vector/buckets/:id/indexes/:idx/query` | Similarity search |

### Utility

| Method | Path | Operation | Auth |
|--------|------|-----------|------|
| GET | `/health` | Health check | None |
| GET | `/status` | Service status | None |
| GET | `/version` | Service version | None |
| DELETE | `/cdn/purge` | Purge CDN cache | Bearer JWT |

## Worked Example -- Create Signed URL

**Request:**
```http
POST /object/sign/avatars/folder/cat.png HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json

{
  "expiresIn": 60000,
  "transform": {
    "width": 200,
    "height": 200,
    "resize": "cover"
  }
}
```

**Response:**
```json
{
  "signedURL": "/object/sign/avatars/folder/cat.png?token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1cmwiOiJhdmF0YXJzL2ZvbGRlci9jYXQucG5nIiwiaWF0IjoxNjE3NzI2MjczLCJleHAiOjE2MTc3MjcyNzN9.s7Gt8ME80iREVxPhH01ZNv8oUn4XtaWsmiQ5csiUHn4"
}
```

The signed URL embeds a JWT token with the object path claim. When `transform` is provided and image transformation is enabled for the tenant, transformation parameters are baked into the signed token. The URL can then be used with `GET /object/sign/:token/:bucket/*` without any auth header.

## Related

- [[SYS-STORAGE]] -- parent system
- [[SCH-STORAGE]] -- data model these endpoints operate on
