# supabase-reef

A structured knowledge base covering 5 Supabase services across 5 source repos — built with [Reef](https://github.com/eunji-jessi-jung/reef), a Claude Code plugin that generates persistent knowledge layers from source code.

## What's in here

69 interlinked markdown artifacts documenting Supabase's architecture:

| Type | Count | What they capture |
|------|-------|-------------------|
| Systems (SYS) | 5 | Service boundary overviews |
| Schemas (SCH) | 5 | Data models with field-level detail |
| APIs (API) | 6 | API surfaces and endpoint inventories |
| Processes (PROC) | 25 | Entity lifecycles, auth flows, error handling |
| Decisions (DEC) | 6 | Architecture decisions and their rationale |
| Contracts (CON) | 4 | Cross-service integration contracts |
| Patterns (PAT) | 7 | Shared patterns across services |
| Risks (RISK) | 5 | Known gaps and backlog candidates |
| Glossary | 7 | Domain terminology per service |

### Services covered

| Service | Language | Source repo |
|---------|----------|-------------|
| Auth | Go | [supabase/auth](https://github.com/supabase/auth) |
| Realtime | Elixir | [supabase/realtime](https://github.com/supabase/realtime) |
| Storage | TypeScript | [supabase/storage](https://github.com/supabase/storage) |
| Studio | React/Next.js | [supabase/supabase](https://github.com/supabase/supabase) (apps/studio, packages/api-types, packages/pg-meta) |
| Client SDK | TypeScript | [supabase/supabase-js](https://github.com/supabase/supabase-js) |

## How this was built

This reef was created by a product manager with no prior knowledge of Supabase's internals. No Go, Elixir, or TypeScript code was read manually. The process:

1. Point Reef at the 5 source repos
2. Run the automated pipeline: `init` → `source` + `snorkel` → `scuba` → partial `deep`
3. Feed in official Supabase documentation as context files
4. Answer Reef's domain questions during the scuba phase

Total time: ~1.5 hours of pipeline execution + ~30 minutes of Q&A.

## Benchmark

We ran 28 agent sessions to test whether this reef actually helps AI agents: 7 tasks x 2 conditions (with/without reef) x 2 models (Claude Opus 4.6, Claude Sonnet 4.6).

|  | Opus (No Reef) | Opus (Reef) | Sonnet (No Reef) | Sonnet (Reef) |
|--|----------------|-------------|------------------|---------------|
| Tokens | 377,924 | 387,183 (+2%) | 400,416 | 388,624 (-3%) |
| Tool calls | 185 | 164 (-11%) | 236 | 173 (-27%) |
| Time | 763s | 654s (-14%) | 954s | 748s (-22%) |
| Quality (rubric) | 49/55 (89%) | 55/55 (100%) | 51/55 (93%) | 55/55 (100%) |

**Key finding:** Sonnet with reef outperformed Opus without reef on our quality rubric (100% vs. 89%), at comparable token cost and speed. Reef helps less capable models more — Sonnet saw 27% fewer tool calls with reef vs. 11% on Opus.

**Caveats:** 7 tasks on 1 codebase. All analysis tasks (tracing, diagnosis, design), not implementation tasks. Creator-designed rubric. Single runs per condition. Read the [full report with all caveats](reef-benchmark-report.md) before drawing conclusions.

## Try it yourself

The best way to evaluate this reef is to use it:

1. Clone this repo
2. Clone the 5 Supabase source repos alongside it
3. Ask an AI agent a cross-service question (e.g., "What breaks if we change the JWT claim format?") — once with the reef in context, once without
4. Compare the results

## How to browse

- **Start with** [`index.md`](index.md) — the full catalog of all 69 artifacts
- **For cross-service understanding**, read the contracts: `artifacts/contracts/`
- **For architecture decisions**, read: `artifacts/decisions/`
- **For "what could go wrong"**, read the risks: `artifacts/risks/`
- **Works in [Obsidian](https://obsidian.md/)** — all cross-references use `[[wikilink]]` syntax

## Built with Reef

[Reef](https://github.com/eunji-jessi-jung/reef) is a Claude Code plugin that builds structured knowledge layers from source code. It reads your repos, extracts architecture, asks you the questions code alone can't answer, and produces interlinked markdown artifacts that AI agents can work from.

This repo is a demo of what Reef produces. It represents a floor — built by a non-engineer with zero Supabase domain knowledge — not a ceiling.
