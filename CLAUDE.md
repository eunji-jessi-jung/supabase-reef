# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

This is a **reef wiki** — a structured knowledge base documenting the Supabase platform architecture. It is NOT a code repository. It contains markdown artifacts that capture systems, APIs, schemas, processes, decisions, patterns, contracts, risks, and glossaries across five Supabase services: Auth (Go), Realtime (Elixir), Storage (TypeScript), Studio (React/Next.js), and Client SDK (supabase-js).

The actual source repos live outside this directory (see `.reef/project.json` for paths).

## Repo Structure

- `index.md` — Auto-generated catalog of all artifacts (do not edit manually)
- `log.md` — Chronological record of reef evolution events
- `artifacts/` — The knowledge base, organized by artifact type:
  - `systems/` — Service boundary overviews (SYS-*)
  - `apis/` — API surface documentation (API-*)
  - `schemas/` — Data model descriptions (SCH-*)
  - `processes/` — Lifecycle and flow documentation (PROC-*)
  - `decisions/` — Architecture decision records (DEC-*)
  - `contracts/` — Cross-service integration contracts (CON-*)
  - `patterns/` — Shared cross-service patterns (PAT-*)
  - `risks/` — Known gaps and backlog candidates (RISK-*)
  - `glossary/` — Domain terminology (GLOSSARY-*)
- `sources/` — Extracted reference material from source repos (schemas, infra, context, raw READMEs)
- `.reef/` — Internal state: project config, questions, manifests, source indexes

## Reef Skills (Slash Commands)

Use these to work with the reef:

- `/reef:help` — Show available skills and recommended next action
- `/reef:artifact` — Explore a topic, capture knowledge, or update an artifact
- `/reef:update` — Re-index sources and update stale artifacts
- `/reef:health` — Validate artifacts and report freshness
- `/reef:lint` — Lint artifacts for format and structural errors
- `/reef:test` — Test the reef against your question bank
- `/reef:feed` — Scan for new context files and connect them to the reef
- `/reef:snorkel` — Auto-discover draft artifacts from sources
- `/reef:scuba` — Manifest-driven artifact generation (deepens snorkel output)
- `/reef:source` — Extract API specs and ERDs from source repos
- `/reef:deep` — Exhaustive line-by-line tracing of critical areas

## Artifact Conventions

Each artifact is a markdown file with YAML frontmatter containing: `id`, `type`, `title`, `domain`, `status` (draft/active), `last_verified`, `freshness_triggers` (source files that invalidate it), and `known_unknowns`.

Artifact IDs follow the pattern `TYPE-DOMAIN[-DETAIL]` (e.g., `SYS-AUTH`, `PROC-STORAGE-OBJECTS-LIFECYCLE`). Cross-references use wiki-link syntax: `[[ARTIFACT-ID]]`.

## Key Relationships

Auth is the trust root — it issues JWTs consumed by Storage (RLS delegation), Realtime (WebSocket auth), and the Client SDK (token injection via `fetchWithAuth`). Studio proxies requests to all services through Next.js API routes (BFF pattern). The Client SDK composes sub-clients (auth-js, postgrest-js, realtime-js, storage-js, functions-js) into a unified interface.
