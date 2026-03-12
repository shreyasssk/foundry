# Foundry — Architecture & Technical Reference

> This document covers the internals of Foundry. For usage and examples, see [README.md](README.md).

---

## How It Works

```
Crucible (planning)  →  Forge (execution)
─────────────────       ─────────────────
Task description        plan.md (always)
Architecture doc   →    design-doc.md (large tasks only)
Codebase context        architecture doc (large tasks only)
                        ↓
                        Committed & pushed code
                        (user creates PR)
```

---

## Crucible — Phase Breakdown

| Phase | Name | Description |
|-------|------|-------------|
| 1 | **Intake** | Resume check → collect task, docs, output dir → assess complexity (small/large) |
| 1.5 | **Validation** | If existing plan/design doc provided, validate against Forge requirements before refining |
| 2 | **Context** | Read inputs, assemble shared context packet (includes complexity) |
| 3 | **Fleet Dispatch** | 3 models draft plan (+ design doc if large) via `task(agent_type="foundry/plan-drafter")` and optionally `task(agent_type="foundry/design-drafter")` |
| 4 | **Convergence** | RALPH loop — cross-review with plain-language convergence checks until all 3 agree (max 10 rounds) |
| 5 | **Merge & Validate** | Merge, validate against Forge's required fields |
| 6 | **Output** | Write final files, cleanup intermediates |
| 7 | **Actions** | Hand off to Forge or done |

### Crucible Output
- `plan.md` — Splits, file breakdowns, dependencies (DAG), acceptance criteria, **complexity classification**
- `design-doc.md` — Problem, solution, types, APIs, error handling, alternatives **(large tasks only)**

---

## Forge — Phase Breakdown

| Phase | Name | Description |
|-------|------|-------------|
| 1 | **Gather** | Resume check → collect document locations |
| 2 | **Locate** | Find and read all documents (architecture doc required for large tasks) |
| 3 | **Analyze** | Extract requirements, validate plan, generate summaries |
| 4 | **Readiness** | Structured report of gaps and warnings |
| 5 | **Confirm** | Final execution preview with user approval |
| 6 | **Execute** | RALPH loop per split + per-split deep review + per-split build gate + commit |
| 7 | **Summary** | Summary and Cleanup |

---

## The RALPH Loop

**RALPH** (named after Ralph Wiggum) — an orchestrator pattern for persistent, self-correcting iteration. Allocate resources with backing specs, give them a goal, loop until done.

> "Software is clay on the pottery wheel. If something isn't right, throw it back on the wheel."

- **In Crucible**: 3 AI models cross-review each other's plans until they converge (plain-language agreement checks)
- **In Forge**: Code agents write code → verifiers check it → issues feed back into next iteration → repeat until all verifiers pass

Each iteration is logged in `forge-state.md` (the loop's persistent memory), enabling resume from any point.

---

## Agents

All agent dispatches use explicit `task(agent_type="foundry/<name>")` calls.

### Crucible Agents
| Agent | Dispatch | Purpose |
|-------|----------|---------|
| `plan-drafter` | `task(agent_type="foundry/plan-drafter")` | Generates/revises Forge-compatible plan.md |
| `design-drafter` | `task(agent_type="foundry/design-drafter")` | Generates/revises Forge-compatible design-doc.md |

### Forge Agents
| Agent | Dispatch | Purpose |
|-------|----------|---------|
| `code-agent` | `task(agent_type="foundry/code-agent")` | Per-file code implementation |
| `plan-verifier` | `task(agent_type="foundry/plan-verifier")` | Checks code against plan (every iteration) |
| `design-verifier` | `task(agent_type="foundry/design-verifier")` | Checks code against design doc (at split completion, large tasks only) |
| `architecture-verifier` | `task(agent_type="foundry/architecture-verifier")` | Checks code against architecture (at split completion, large tasks only) |
| `scribe` | `task(agent_type="foundry/scribe")` | Conditional task logging |

> **Note:** The deep review step in Forge Phase 6 uses external agents from the **deep-review** plugin (`deep-review/architect`, `deep-review/advocate`, `deep-review/skeptic`) — these are NOT foundry agents.

---

## Models Used (Crucible Fleet)

| Model | ID | Strength |
|-------|-----|----------|
| Claude Opus | `claude-opus-4.6` | Deep reasoning, architecture |
| GPT Codex | `gpt-5.1-codex-max` | Code-first planning |
| Gemini Pro | `gemini-3-pro-preview` | Breadth, alternatives (**must use `mode="sync"` — fails on background dispatch**) |

> **Gemini sync constraint:** This constraint applies to **Crucible fleet dispatch only** (when dispatching plan-drafter/design-drafter agents with Gemini). Forge code-agent dispatch is unaffected — code agents use the default model and don't receive a model parameter.

---

## Task Slug Algorithm

The task slug is derived from the **task name** (the plan's `# ` title or `## Overview` first sentence) and used as the directory name under `~/.copilot/foundry/`:

1. Take the task name (e.g., `Add OAuth2 Authentication`)
2. Convert to lowercase
3. Replace spaces and special characters with hyphens (`-`)
4. Remove any characters not matching `[a-z0-9\-]`
5. Collapse consecutive hyphens into one
6. Trim leading/trailing hyphens
7. Truncate to **50 characters** (break at the last whole word that fits; if the first word exceeds 50 chars, hard-truncate at 50)
8. Result: `add-oauth2-authentication`

> **Canonical source:** Both Crucible and Forge derive the slug from the task name using this algorithm. The slug is recorded in `crucible-state.md` and `forge-state.md` so downstream consumers never need to re-derive it.

This slug is also used for checkpoint tags, forge-state references, and the shared working directory.

**Checkpoint tag format:** `forge-checkpoint--<slug>-split-<N>-iter-<I>` where:
- `<slug>` — task slug derived above (branch `/` replaced with `-` to avoid git ref conflicts)
- `<N>` — split number (1-indexed; omitted for single-branch mode)
- `<I>` — iteration number within the split (1-indexed)
- Example: `forge-checkpoint--add-oauth2-authentication-split-2-iter-3`

---

## Task Complexity

Crucible assesses task complexity during intake and classifies tasks as **small** or **large**:

| Aspect | Small Task | Large Task |
|--------|-----------|------------|
| Files | ≤ 5 files | > 5 files or multiple components |
| Splits | Single split likely | Multiple splits needed |
| Scope | Bug fix, config change, simple feature | New APIs, services, data models |
| Design doc | Skipped | Generated |
| Architecture doc | Not required by Forge | Required by Forge |
| Verifiers | Plan only | Plan + architecture + design |

The classification is stored in the plan's `## Complexity` section and read by Forge to adjust ceremony automatically.

---

## Split Strategy

After complexity assessment, Crucible evaluates whether the task needs multiple split branches or can be done on a single branch:

| Indicator | Single Branch | Grey Zone (6–7 files) | Multi-Split |
|-----------|--------------|----------------------|-------------|
| Scope | Tightly coupled changes | Depends on coupling | Distinct phases or layers |
| Files | ≤ 5 files | 6–7 files (user decides) | > 8 files or independent groups |
| Dependencies | All files interdependent | Evaluate case-by-case | Clear dependency ordering |
| Review | One review pass sufficient | Present both options | Staged review preferred |

### How It Works

1. **Crucible** analyzes the task and proposes `single` or `multi` — prompts user for confirmation
2. **Plan** stores the decision in `## Execution Config` as `Split Strategy: single | multi`
3. **Forge** reads the strategy and adjusts branch creation, push, and summary accordingly

### Single-Branch Mode
- Branch: `<task-branch>` directly (no `/split-N` suffix)
- One branch, one eventual PR
- Plan has exactly 1 split in `## Splits`
- Skip split relationship question (irrelevant for single-branch)
- Simpler completion summary with single branch table

### Multi-Split Mode (default)
- Branch: `<task-branch>/split-N` per split, chained from previous
- Multiple branches, merge in order
- Plan has N splits in `## Splits`
- Split relationship (chained/independent) required

---

## Safety Features

- **Adaptive convergence** (Crucible) — plain-language convergence checks, never forces premature consensus
- **Hard caps** — 10 rounds in Crucible, 10 iterations per split in Forge, 5 rounds of deep review **per split**
- **Resume support** — both skills track state and can pick up where they left off
- **Per-split build gate** (Forge) — full build per split after deep review passes, before commit
- **Git safety** (Forge) — fetch/rebase before push, specific file staging, user-specified base branch
- **Architecture required** (Forge) — architecture doc is mandatory for large tasks; small tasks skip it per Crucible's complexity assessment
- **Local deep review** (Forge) — runs against branch diff, no PR needed
- **Clean intermediates** — working files deleted, only outputs survive

---

## Key Files Generated

### Shared Working Directory (`~/.copilot/foundry/<task-slug>/`)

Both Crucible and Forge share one directory per task. Crucible creates it first; Forge reuses it.

**Crucible files:**
| File | Purpose | Persists? |
|------|---------|-----------|
| `crucible-state.md` | Resume state (round, models, complexity) | Deleted on completion |
| `crucible-round-*.md` | Intermediate model outputs | Deleted on completion |
| `plan.md` | Final merged plan | ✅ Output |
| `design-doc.md` | Final merged design doc (large only) | ✅ Output |

**Forge files (same directory):**
| File | Purpose | Persists? |
|------|---------|-----------|
| `forge-state.md` | Resume state (split, iteration, branch, complexity, split strategy) | Deleted on completion |
| `forge-coordination.md` | Consolidated verifier feedback | Deleted on completion |
| `forge-verifier-*.md` | Individual verifier results | Deleted on completion |
| `forge-diff-*.patch` | Per-file diffs for verifiers | Deleted on completion |
| `forge-summary-*.md` | Document summaries for code agents | Deleted on completion |
| `forge-task-log.md` | Scribe's iteration-by-iteration log | ✅ Output |
| `forge-summary.md` | Execution summary — branches, splits, merge order, next steps | ✅ Output |

> **Zero repo pollution:** All Foundry files live outside the repo. Only code changes touch the source directory.

---

## Version History

- **v1.6.0** (2025-03) — Split strategy assessment: Crucible evaluates whether a task can be done on a single branch or needs multi-split branching. Stores `Split Strategy: single | multi` in plan's `## Execution Config`. Forge reads strategy and adjusts branch creation (no `/split-N` for single), push commands, rebase targets, forge-state template, task log header, completion summary, and user-facing report. Single-branch mode simplifies the flow for small/tightly-coupled tasks. Plan-drafter template updated. Hard rules updated to respect single-branch mode. Additional safety: execution config input validation (regex guards against command injection), DAG validation via Kahn's algorithm, checkpoint tag scoping to current task, build-fix hard cap of 5, deep review diff encoding with PS5.1 fallback, coordination file archiving with delimiter counting.
- **v1.5.0** — Headless Forge: branch config (base branch, prefix, split relationship) moved from Forge Phase 5 to Crucible intake. Plan template includes `## Execution Config` section. Forge reads config from plan and runs headless after one "shall I proceed?" confirmation. Fallback prompting only if plan lacks execution config. Plan-drafter and plan-verifier updated for new section. README restructured: user-facing with examples, internals moved to ARCHITECTURE.md.
- **v1.4.3** — Round 13 doc fixes: README Phase 1.5 aligned with SKILL.md (Phase 1 includes complexity, 1.5 = Validation), Phase 5 preview now complexity-aware for verifier text, scribe dispatch includes complexity field, deep-review fix loop verifier scope clarified, "two decisions" → "three", README Phase 2 arch doc qualified for large tasks, coordination file template covers small-task skip, Crucible context packet complexity instruction references plan-drafter template
- **v1.4.2** — Round 12 fixes: Forge readiness now blocks on missing ## Complexity section and missing design doc for large tasks; Crucible Phase 5-6 fully complexity-aware (validation/cross-check/output/cleanup/hand-off all conditional on small vs large); Phase 6 design verifier skip path tightened for large tasks
- **v1.4.1** — Complexity flow fixes (round 11 review): format mismatch between plan template (`Classification: small`) and Forge detection fixed, code-agent spec updated for conditional summaries, Phase 7 cleanup made dynamic (only lists existing files), resume backward compat for pre-v1.4.0 states, Crucible validation checks for ## Complexity section, convergence logic handles plan-only mode, plan-verifier checks complexity section integrity, README qualified for small-task paths, log message punctuation standardized
- **v1.4.0** — Task complexity assessment: Crucible classifies tasks as small/large, recommends design doc decision. Forge reads complexity flag — skips design doc, architecture doc, and their verifiers for small tasks. Plan template includes `## Complexity` section. Scribe logs complexity and skipped verifiers. Caution messages shown when ceremony is reduced.
- **v1.3.6** — Doc/state polish: README reflects user-prompted base branch, forge-state template uses dynamic chained flag, legacy state resume asks for missing base-branch, Phase 6 guard prevents independent splits from chaining
- **v1.3.5** — User-prompted base branch (no auto-detection), split relationship check (chained vs independent), base-branch persisted in forge-state.md, removed all DEFAULT_BRANCH auto-detection code
- **v1.3.4** — Default branch fallback (origin/HEAD → main → master), BOM-free UTF-8 patches (utf8NoBOM + PS 5.1 fallback), .gitignore upgrade detection, archive + diff patch cleanup, branch preference moved to Phase 5, Crucible file discovery uses git ls-files with 500 limit
- **v1.3.3** — Resume-safe branch ordering (check existing before creating), user branch prefix preference prompt, UTF-8 patch encoding on PowerShell, checkpoint tag cleanup at completion, per-file diff patch cleanup, origin parent fallback for fresh-env resume
- **v1.3.2** — Cross-platform hardening: PowerShell-safe branch creation (no `/dev/null`), slugified checkpoint tags (no git ref path conflicts), remote-tracking fallback on resume, large-diff chunking for deep review, hard-cap extension persisted to state, origin fallback for parent-split branches
- **v1.3.1** — Per-split branching fixes: deep review rebase targets, namespaced checkpoint tags, resume-safe branch creation, collision prevention
- **v1.3.0** — Per-split branching: each split gets its own branch (`<task>/split-N`), chained from the previous split for ordered review and merge
- **v1.2.3** — Final polish: typo fix, scribe entry naming, deep review cap in README, design verifier skip logic, consistent RALPH wording
- **v1.2.2** — Doc polish: aligned phase numbering, clarified build gate flow, unified dependency severity, consistent split limits, typo fixes
- **v1.2.1** — Cleanup: removed per-split build gate remnant, unified rollback to checkpoint tags, cross-platform commands in Forge, plain-language convergence throughout, removed dev markers
- **v1.2.0** — No PR creation (push-only), build gate moved post-execution, local deep review, architecture doc required, explicit agent dispatch, plain-language convergence checks
- **v1.1.0** — Fixed all 10 issues from multi-model evaluation (infinite loop protection, structural convergence, cross-platform, safe cleanup, rollback/retry, and more)
- **v1.0.0** — Initial release combining Crucible + Forge under one plugin
