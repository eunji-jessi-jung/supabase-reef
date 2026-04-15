---
id: "SCH-STUDIO-PG-META"
type: "schema"
title: "Pg-Meta Postgres Introspection Model"
domain: "studio"
status: "draft"
last_verified: 2026-04-15
freshness_note: "snorkel-depth scan"
freshness_triggers:
  - "packages/pg-meta/src/"
known_unknowns:
  - "Exact SQL queries used for each introspection type"
  - "Performance characteristics of introspection queries on large databases"
  - "How pg-meta handles schema versioning across different Postgres versions"
tags:
  - studio
  - postgres
  - introspection
aliases:
  - "pg-meta"
relates_to:
  - type: "parent"
    target: "[[SYS-STUDIO]]"
sources:
  - category: "implementation"
    type: "github"
    ref: "packages/pg-meta/src/index.ts"
notes: ""
---

## Overview

pg-meta is a TypeScript library that introspects Postgres system catalogs (information_schema, pg_catalog) to provide structured metadata about database objects. It is the backbone of Studio's database explorer, table editor, and policy management interfaces. Each Postgres object type has its own module with queries, types, and helpers.

## Key Facts

- 16 introspection modules covering major Postgres object types → `packages/pg-meta/src/`
- Each module (e.g., pg-meta-tables.ts) provides list/get/create/update/delete operations → `packages/pg-meta/src/`
- Column privileges and table privileges tracked separately for fine-grained access display → `packages/pg-meta/src/pg-meta-column-privileges.ts`
- Version detection for handling Postgres version differences → `packages/pg-meta/src/pg-meta-version.ts`
- Comprehensive test suite mirroring the module structure → `packages/pg-meta/test/`

## Entities

These are not database tables in the traditional sense — they are Postgres system catalog objects that pg-meta reads.

| Introspection Module | Postgres Objects | Description |
|---------------------|-----------------|-------------|
| pg-meta-tables | Tables | List, create, update, delete tables with full column/index info |
| pg-meta-columns | Columns | Column definitions, types, defaults, constraints |
| pg-meta-functions | Functions | Stored procedures and functions with argument types |
| pg-meta-policies | Policies | RLS policies with their expressions |
| pg-meta-roles | Roles | Database roles and their privileges |
| pg-meta-views | Views | Regular views with definitions |
| pg-meta-materialized-views | Materialized Views | Materialized views with refresh metadata |
| pg-meta-indexes | Indexes | Index definitions, types, and columns |
| pg-meta-extensions | Extensions | Installed Postgres extensions |
| pg-meta-triggers | Triggers | Trigger definitions and timing |
| pg-meta-schemas | Schemas | Database schemas |
| pg-meta-types | Types | Custom types and enums |
| pg-meta-publications | Publications | Logical replication publications |
| pg-meta-foreign-tables | Foreign Tables | Foreign data wrapper tables |
| pg-meta-config | Config | Postgres configuration parameters |
| pg-meta-column-privileges | Column Privileges | Per-column grant information |
| pg-meta-table-privileges | Table Privileges | Per-table grant information |

## Related

- [[SYS-STUDIO]] -- parent system
- [[API-STUDIO]] -- API exposing pg-meta capabilities
