---
id: "DEC-MULTI-PROTOCOL-STORAGE"
type: "decision"
title: "Storage Multi-Protocol Design Decision"
domain: "storage"
status: "draft"
last_verified: 2026-04-15
freshness_note: "snorkel-depth scan"
freshness_triggers:
  - "src/http/routes/index.ts"
  - "src/storage/protocols/"
known_unknowns:
  - "Rationale not available from code alone"
  - "Timeline for when each protocol was added"
  - "Usage distribution across protocols in production"
tags:
  - decision
  - architecture
aliases: []
relates_to:
  - type: "parent"
    target: "[[SYS-STORAGE]]"
sources:
  - category: "implementation"
    type: "github"
    ref: "src/storage/protocols/s3/s3-handler.ts"
notes: ""
---

## Context

Storage supports four distinct access protocols: HTTP/REST (the original and primary interface), TUS (resumable uploads), S3-compatible API, and Iceberg REST Catalog. All protocols share the same underlying storage engine and Postgres metadata layer.

## Decision

Implement multiple access protocols sharing a single storage engine rather than operating separate services per protocol. Each protocol handler translates protocol-specific semantics into operations on the common Storage class.

## Key Facts

- Four protocol handlers share one Storage engine → `src/storage/protocols/`, `src/http/routes/`
- S3 protocol handler is the most complex, implementing core S3 operations with 5GB max part size → `src/storage/protocols/s3/s3-handler.ts`
- TUS protocol enables resumable uploads for unreliable connections → `src/storage/protocols/tus/`
- Iceberg provides data lakehouse access patterns → `src/http/routes/iceberg/`
- Vector protocol adds AI/ML embedding storage capabilities → `src/http/routes/vector/`

## Rationale

Not available from code alone. Likely driven by customer needs: S3 compatibility for existing tooling, TUS for large file uploads, Iceberg for data engineering workflows.

## Consequences

### Positive
- Single metadata layer means consistent authorization (RLS) across all protocols
- Shared storage engine reduces code duplication and maintenance burden
- Users can access the same files through whichever protocol suits their use case

### Negative
- Each protocol adds complexity to the codebase
- Protocol-specific edge cases may interact unpredictably
- S3 compatibility is inherently partial — some S3 features return stubbed responses (e.g., versioning always "Suspended")

### Neutral
- Each new protocol is additive — existing protocols remain stable

## Related

- [[SYS-STORAGE]] -- parent system
- [[API-STORAGE]] -- API surface across all protocols
