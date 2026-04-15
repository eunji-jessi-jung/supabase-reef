# Reef Benchmark Report: Does a Knowledge Layer Actually Help AI Agents?

**Date:** 2026-04-15
**Test subject:** supabase-reef (knowledge base covering 5 Supabase services across 5 source repos)
**Models tested:** Claude Opus 4.6, Claude Sonnet 4.6
**Method:** 7 tasks x 2 conditions (with/without reef) x 2 models = 28 total agent runs. Same question, same repos, rubric-scored.

---

## Executive Summary

We ran 28 agent sessions to test whether a reef — a structured knowledge layer built from source code — measurably improves AI agent performance. The short answer: **yes, but the advantage is uneven.** Reef helps most on hard cross-service reasoning tasks and helps less capable models more than strong ones.

|  | Opus (No Reef) | Opus (Reef) | Sonnet (No Reef) | Sonnet (Reef) |
|--|----------------|-------------|------------------|---------------|
| **Total tokens** | 377,924 | 387,183 (+2%) | 400,416 | 388,624 (-3%) |
| **Total tool calls** | 185 | 164 (-11%) | 236 | 173 (-27%) |
| **Total time** | 763s | 654s (-14%) | 954s | 748s (-22%) |
| **Rubric score** | 49/55 (89%) | 55/55 (100%) | 51/55 (93%) | 55/55 (100%) |

The most interesting finding: **Sonnet with reef outperformed Opus without reef on our rubric** (100% vs. 89%), at comparable token cost and speed. If this holds, reef could serve as a cost optimization — use a cheaper model with structured context instead of a more expensive model without it.

But this benchmark has real limitations. Read the [Caveats](#caveats-read-before-citing-any-of-this) section before drawing conclusions.

---

## Test Design

### The 7 Tasks

| # | Task | Difficulty | What it tests |
|---|------|-----------|---------------|
| 1 | Trace JWT from client SDK to Storage authorization | Moderate | Cross-service flow (2 services) |
| 2 | Explain Realtime's channel authorization mechanism | Moderate | Deep single-service understanding |
| 3 | Design a webhook system for Storage | Moderate | Single-service design |
| 4 | Diagnose why CDC breaks after email update | Hard | Cross-service state reasoning (Auth + Realtime) |
| 5 | Design unified audit log across 3 services | Hard | Cross-cutting implementation (Go + TS + Elixir) |
| 6 | Analyze blast radius of JWT format change | Hard | Impact analysis across all 5 services |
| 7 | Architecture plan for collaborative editor | Hard | End-to-end design (Realtime + Storage + Auth) |

### Quality Rubric

Each task has 7-8 specific checkpoints scored pass/fail — concrete, verifiable findings, not subjective judgments. Examples:

- **Task 1:** "Did the agent find the pre-flight permission check (canUpload/testPermission with rollback)?"
- **Task 4:** "Did the agent identify that confirm_token updates in-memory claims but NOT the DB subscription?"
- **Task 6:** "Did the agent flag that user-written RLS policies can't be auto-migrated?"

Full rubric in Appendix A.

---

## Results: Opus

### Efficiency

| Task | Tokens (No Reef) | Tokens (Reef) | Tool Calls (No Reef) | Tool Calls (Reef) | Time (No Reef) | Time (Reef) |
|------|------------------|---------------|---------------------|-------------------|----------------|-------------|
| 1. Storage auth flow | 51,421 | 51,570 | 19 | 18 | 76s | **62s** |
| 2. Realtime auth | 58,708 | **51,921** | 20 | **13** | 108s | **67s** |
| 3. Webhook design | **34,473** | 45,249 | 19 | 19 | **67s** | 79s |
| 4. CDC bug diagnosis | 72,908 | **53,075** | 34 | **28** | 175s | **129s** |
| 5. Audit log design | **41,722** | 65,285 | 30 | 30 | **102s** | 111s |
| 6. JWT blast radius | 61,606 | **53,926** | 31 | **30** | 123s | **113s** |
| 7. Collab editor | 57,086 | 66,157 | 32 | **26** | 112s | **93s** |
| **Total** | **377,924** | **387,183** | **185** | **164** | **763s** | **654s** |

### Quality

| Task | No Reef | Reef | What No-Reef Missed |
|------|---------|------|---------------------|
| 1. Storage auth flow | 6/8 | 8/8 | Pre-flight permission check, two-phase auth pattern |
| 2. Realtime auth | 6/8 | 8/8 | Periodic 5-min re-validation timer, mid-session token refresh |
| 3. Webhook design | 7/7 | 7/7 | (tie) |
| 4. CDC bug diagnosis | 8/8 | 8/8 | (tie) |
| 5. Audit log design | 8/8 | 8/8 | (tie) |
| 6. JWT blast radius | 7/8 | 8/8 | User-written RLS policies can't be auto-migrated |
| 7. Collab editor | 7/8 | 8/8 | No delivery guarantee warning |
| **Total** | **49/55 (89%)** | **55/55 (100%)** | |

**On Opus, reef's efficiency advantage is moderate** — 14% faster, 11% fewer tool calls, flat on tokens. The quality gap is real but narrow: Opus without reef already scores 89%. Reef catches operational details and cross-service implications that Opus misses when reading code alone, but Opus is strong enough to find most things through exploration.

---

## Results: Sonnet

### Efficiency

| Task | Tokens (No Reef) | Tokens (Reef) | Tool Calls (No Reef) | Tool Calls (Reef) | Time (No Reef) | Time (Reef) |
|------|------------------|---------------|---------------------|-------------------|----------------|-------------|
| 1. Storage auth flow | 55,399 | **39,978** | 37 | **11** | 117s | **59s** |
| 2. Realtime auth | **56,609** | 45,337 | **12** | 17 | **68s** | 102s |
| 3. Webhook design | **42,490** | 45,901 | 45 | **20** | 150s | **86s** |
| 4. CDC bug diagnosis | **52,504** | 76,348 | **35** | 34 | **137s** | 153s |
| 5. Audit log design | 68,309 | **57,749** | **33** | 38 | **126s** | 135s |
| 6. JWT blast radius | 56,265 | **47,430** | 36 | **26** | 200s | **99s** |
| 7. Collab editor | 68,840 | 75,881 | 38 | **27** | 156s | **114s** |
| **Total** | **400,416** | **388,624** | **236** | **173** | **954s** | **748s** |

### Quality

| Task | No Reef | Reef | What No-Reef Missed |
|------|---------|------|---------------------|
| 1. Storage auth flow | 8/8 | 8/8 | (tie — Sonnet found pre-flight check without reef) |
| 2. Realtime auth | 6/8 | 8/8 | Periodic 5-min re-validation timer, mid-session token refresh |
| 3. Webhook design | 7/7 | 7/7 | (tie) |
| 4. CDC bug diagnosis | 8/8 | 8/8 | (tie) |
| 5. Audit log design | 8/8 | 8/8 | (tie) |
| 6. JWT blast radius | 7/8 | 8/8 | User-written RLS policies can't be auto-migrated |
| 7. Collab editor | 7/8 | 8/8 | No delivery guarantee warning |
| **Total** | **51/55 (93%)** | **55/55 (100%)** | |

**On Sonnet, reef's efficiency advantage is larger** — 22% faster, 27% fewer tool calls, and a slight token savings (-3%). Sonnet without reef thrashes more than Opus (236 tool calls vs. 185), and reef cuts that thrashing significantly. The quality pattern is similar: same checkpoints missed, same checkpoints caught with reef.

Notably, Sonnet without reef scored *higher* than Opus without reef (93% vs. 89%) — Sonnet happened to find the pre-flight permission check in Task 1 that Opus missed. Single runs have variance. Don't read too much into that.

---

## Cross-Model Comparison

### Reef helps weaker models more

| Metric | Opus Delta | Sonnet Delta |
|--------|-----------|-------------|
| Token cost | +2% | -3% |
| Tool call reduction | -11% | **-27%** |
| Time reduction | -14% | **-22%** |
| Quality gap closed | 89% → 100% | 93% → 100% |

Sonnet sees roughly double the efficiency benefit from reef. This makes sense: Opus compensates for missing context through more capable reasoning and exploration. Sonnet needs more guidance, and reef provides it.

### The cost-efficiency comparison

| | Opus (No Reef) | Sonnet (Reef) |
|--|----------------|---------------|
| Quality score | 49/55 (89%) | 55/55 (100%) |
| Total tokens | 377,924 | 388,624 |
| Total tool calls | 185 | 173 |
| Total time | 763s | 748s |

Sonnet with reef scores higher on our rubric than Opus without reef, at comparable cost and speed. If Sonnet API pricing is lower than Opus, this represents a direct cost savings with no quality sacrifice — on these tasks.

### What both models consistently miss without reef

| Missed insight | Task | Why code alone doesn't surface it |
|---------------|------|----------------------------------|
| Periodic 5-min token re-validation | T2 | Timer interval buried in module attributes, not on any obvious code path |
| Mid-session JWT refresh mechanism | T2 | Requires understanding the full lifecycle, not just the join flow |
| User-written RLS policies can't be auto-migrated | T6 | A strategic insight about the ecosystem, not a code-level fact |
| No delivery guarantee for Realtime messages | T7 | Documented in risk artifacts, not visible in the happy-path code |

These are operational and architectural knowledge — exactly what reef's PROC-, CON-, and RISK- artifacts encode. They're real findings that matter for production systems. But they're also the kind of thing reef is specifically designed to capture, which brings us to the caveats.

---

## Spotlights

### Task 1 on Sonnet — largest efficiency delta

| | Sonnet (No Reef) | Sonnet (Reef) |
|--|------------------|---------------|
| Tokens | 55,399 | 39,978 |
| Tool calls | 37 | 11 |
| Time | 117s | 59s |
| Quality | 8/8 | 8/8 |

Same quality. 70% fewer tool calls. 50% less time. The no-reef agent made 37 tool calls — searching, reading wrong files, backtracking. The reef agent read the relevant artifacts, then made 4 targeted verification reads in source code.

### Task 6 on Sonnet — largest time delta

| | Sonnet (No Reef) | Sonnet (Reef) |
|--|------------------|---------------|
| Tokens | 56,265 | 47,430 |
| Tool calls | 36 | 26 |
| Time | 200s | 99s |
| Quality | 7/8 | 8/8 |

Better quality AND 50% less time. The JWT blast radius task requires finding every claim-reading location across 5 services. Without reef, Sonnet spent over 3 minutes hunting through repos. With reef, contract artifacts immediately pointed to which services consume JWT claims and where.

---

## Caveats (Read Before Citing Any of This)

This benchmark is a useful signal, not a proof. Here's what limits the conclusions you can draw:

**1. Small sample, single codebase.** 7 tasks on 1 codebase (Supabase). Supabase is well-structured, well-documented open-source code. Results may differ on messy internal codebases, monorepos, or codebases in different languages. We don't know yet.

**2. The benchmark was designed by the reef creator.** Task selection, rubric design, and evaluation were all done by the same person who built reef. There's no intentional bias, but there's unavoidable selection bias — the tasks naturally test the kind of knowledge reef is designed to capture.

**3. The rubric may be reef-shaped.** The checkpoints that differentiate reef from no-reef (periodic re-validation timer, user-written RLS risk, no delivery guarantee) are exactly the kind of operational and architectural knowledge reef encodes. A rubric focused on different dimensions — correctness of generated code, compilation success, test passage — might show a smaller gap or no gap at all.

**4. All 7 tasks are analysis tasks, not implementation tasks.** Trace a flow, design a system, diagnose a bug, analyze an impact. Engineers spend significant time on these tasks, but they also spend time writing code, fixing tests, and making PRs. Reef's value on implementation tasks is untested here.

**5. Single runs per condition.** Each task was run once per condition. LLM outputs are stochastic — running the same task again might produce different results. The quality scores (especially the 100% with reef) should be read as "reef made it possible to find these things," not "reef guarantees finding them every time."

**6. The 100% rubric score is a ceiling artifact.** Both models hit 55/55 with reef. This likely means the rubric isn't granular enough to differentiate further, not that reef-equipped agents are perfect. A harder rubric would probably show gaps.

**7. Build cost is not included.** The benchmark measures per-task usage cost. Building the supabase-reef took ~1-2 hours of Opus-tier pipeline execution. To break even on efficiency gains alone, you'd need many agent sessions against the reef. The ROI depends on how frequently a team runs cross-service agent tasks.

**8. The reef was built on a codebase the creator had no prior knowledge of.** This is both a strength (proves the pipeline works without domain expertise) and a limitation (a domain expert's reef might produce different — possibly much better — results). We tested the floor, not the ceiling.

---

## Conclusions

### What we can say

1. **Reef consistently improves agent quality on cross-service reasoning tasks.** Both models missed the same operational and architectural insights without reef and found them with reef. The pattern is consistent, not random.

2. **Reef's efficiency benefit is larger on less capable models.** Sonnet sees 27% fewer tool calls with reef (vs. 11% on Opus). If you're using a cheaper model, reef provides more value.

3. **Sonnet + reef outperformed Opus alone on our rubric.** This is the most practically interesting finding — it suggests reef can be a cost optimization strategy. But see caveats about rubric design and sample size.

4. **Token cost is neutral.** Reef doesn't save or cost more tokens per task. The savings are in wall-clock time and tool call efficiency.

5. **The highest-value artifacts are contracts, processes, and risks.** The insights reef enabled were all about cross-service state, operational behavior, and ecosystem-level implications — not schemas or API surfaces that agents can find from code.

### What we can't say (yet)

- Whether these results generalize to other codebases, languages, or team structures
- Whether reef helps with implementation tasks (writing code, fixing tests) as much as analysis tasks
- Whether the build cost pays for itself over time for a typical team
- Whether a domain expert's reef produces meaningfully better results than our non-expert-built reef
- Whether reef-equipped agents produce better *code*, not just better *analysis*

### The honest position

Reef is a real tool that produces measurable improvements on a specific class of tasks: hard, cross-service reasoning where code alone doesn't surface the full picture. The advantage is moderate on strong models and more significant on cheaper models. It doesn't save tokens, but it saves time and catches things agents miss.

Whether it's worth the build cost depends on how often your team faces these cross-service tasks and how much you rely on AI agents for them. For a team running daily agent sessions against a complex multi-service backend, the math probably works. For occasional use on a small codebase, it probably doesn't.

We're shipping this with the data, the caveats, and the raw numbers. Draw your own conclusions.

---

## Context: How This Reef Was Built

The supabase-reef was created by a product manager with no prior knowledge of Supabase's internals. The creator did not read the Go, Elixir, or TypeScript source code. The process:

1. Point Reef at the 5 source repos
2. Run the automated pipeline: `init` → `source` + `snorkel` → `scuba` → partial `deep`
3. Feed in official Supabase documentation as context files
4. Answer Reef's domain questions during the scuba phase (as a PM, not an engineer)

No manual artifact writing. No code review. This is a floor — a team of domain experts enriching the same reef with institutional knowledge would likely produce stronger results, though we haven't tested that.

---

## Appendix A: Quality Rubric

### Task 1: Storage Auth Flow (8 checkpoints)
1. fetchWithAuth injects Bearer header from session
2. apikey header also set (anon key)
3. JWT plugin extracts token from Authorization header
4. jose library used with multiple algorithm support
5. JWT verification cache (LRU) described
6. setScope() injects GUC variables into Postgres transaction
7. Pre-flight permission check (canUpload/testPermission with rollback)
8. Two-phase authorization (pre-flight + actual write)

### Task 2: Realtime Authorization (8 checkpoints)
1. WebSocket connect validates JWT via tenant secret
2. Channel join confirm_token flow
3. Private channel RLS probe (INSERT + SELECT + ROLLBACK on realtime.messages)
4. set_conn_config sets session variables for RLS
5. CDC per-row RLS via apply_rls SQL function
6. Subscription claims stored in realtime.subscription table
7. Periodic re-validation timer (~5 min)
8. Token refresh mid-session via access_token event

### Task 3: Webhook Design (7 checkpoints)
1. Found pg-boss queue infrastructure
2. Found existing Webhook worker + lifecycle event classes
3. Found existing sendWebhook() call sites in object.ts / uploader.ts
4. Found Auth's Standard Webhooks signing pattern
5. Proposed per-bucket webhook subscription table
6. Proposed extending BaseEvent.sendWebhook() for fan-out
7. Concrete file change list provided

### Task 4: Bug Diagnosis — CDC After Email Update (8 checkpoints)
1. Email update doesn't immediately issue new JWT
2. Claims stored in realtime.subscription at join time
3. apply_rls reads stored claims (not live) for RLS evaluation
4. Broadcast uses in-memory PubSub, not stored claims
5. confirm_token updates in-memory but NOT DB subscription
6. access_token handler DOES call postgres_subscribe()
7. Root cause correctly identified (stale claims in subscription table)
8. Concrete fix proposed at correct code location

### Task 5: Unified Audit Log (8 checkpoints)
1. Found Auth's existing audit log system (NewAuditLogEntry)
2. Found Auth's specific call sites across api/ files
3. Found Storage's write operation locations (object.ts, uploader.ts, storage.ts)
4. Found Storage's pg-boss queue for async delivery
5. Found Realtime's write operations (broadcast handler, presence handler)
6. Identified Realtime has no persistent queue infrastructure
7. Reasonable unified schema design
8. Identified key hard problems (volume, transactions, multi-tenant routing)

### Task 6: JWT Format Change Blast Radius (8 checkpoints)
1. Auth AccessTokenClaims struct identified as token producer
2. Storage jwt.ts role reading identified (payload.role)
3. Storage setScope() / request.jwt.claims as primary breakage vector
4. Realtime required claims check ("role", "exp" at top level)
5. Realtime set_conn_config breakage
6. Client SDK treats tokens as opaque (low direct impact)
7. User-written RLS policies as major uncontrollable risk
8. Migration path with dual-write/dual-emit phase

### Task 7: Collaborative Editor Architecture (8 checkpoints)
1. Presence API (track/presenceState) with correct usage pattern
2. Private channel flag for RLS enforcement
3. Broadcast + Postgres Changes combined for optimistic updates
4. Storage bucket RLS using document_members join pattern
5. Connection/rate limit gotchas with specific numbers
6. Presence payload size limits
7. Token expiry / periodic re-auth handling
8. CDC delivery not guaranteed — don't rely solely on broadcast

---

## Appendix B: Raw Data (All 28 Runs)

| Task | Model | Reef? | Tokens | Tool Calls | Time (s) | Quality |
|------|-------|-------|--------|------------|----------|---------|
| 1 | Opus | No | 51,421 | 19 | 76 | 6/8 |
| 1 | Opus | Yes | 51,570 | 18 | 62 | 8/8 |
| 1 | Sonnet | No | 55,399 | 37 | 117 | 8/8 |
| 1 | Sonnet | Yes | 39,978 | 11 | 59 | 8/8 |
| 2 | Opus | No | 58,708 | 20 | 108 | 6/8 |
| 2 | Opus | Yes | 51,921 | 13 | 67 | 8/8 |
| 2 | Sonnet | No | 56,609 | 12 | 68 | 6/8 |
| 2 | Sonnet | Yes | 45,337 | 17 | 102 | 8/8 |
| 3 | Opus | No | 34,473 | 19 | 67 | 7/7 |
| 3 | Opus | Yes | 45,249 | 19 | 79 | 7/7 |
| 3 | Sonnet | No | 42,490 | 45 | 150 | 7/7 |
| 3 | Sonnet | Yes | 45,901 | 20 | 86 | 7/7 |
| 4 | Opus | No | 72,908 | 34 | 175 | 8/8 |
| 4 | Opus | Yes | 53,075 | 28 | 129 | 8/8 |
| 4 | Sonnet | No | 52,504 | 35 | 137 | 8/8 |
| 4 | Sonnet | Yes | 76,348 | 34 | 153 | 8/8 |
| 5 | Opus | No | 41,722 | 30 | 102 | 8/8 |
| 5 | Opus | Yes | 65,285 | 30 | 111 | 8/8 |
| 5 | Sonnet | No | 68,309 | 33 | 126 | 8/8 |
| 5 | Sonnet | Yes | 57,749 | 38 | 135 | 8/8 |
| 6 | Opus | No | 61,606 | 31 | 123 | 7/8 |
| 6 | Opus | Yes | 53,926 | 30 | 113 | 8/8 |
| 6 | Sonnet | No | 56,265 | 36 | 200 | 7/8 |
| 6 | Sonnet | Yes | 47,430 | 26 | 99 | 8/8 |
| 7 | Opus | No | 57,086 | 32 | 112 | 7/8 |
| 7 | Opus | Yes | 66,157 | 26 | 93 | 8/8 |
| 7 | Sonnet | No | 68,840 | 38 | 156 | 7/8 |
| 7 | Sonnet | Yes | 75,881 | 27 | 114 | 8/8 |
