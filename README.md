# Foundry — Complete AI Task Lifecycle

> The foundry contains both the crucible (where plans are refined) and the forge (where code is shaped).

## Two Skills, One Workflow

| Skill | Trigger | What It Does |
|-------|---------|-------------|
| **foundry:crucible** | `/crucible`, "plan this", "design this task" | Multi-model plan refinery — 3 AI models converge on a Forge-compatible plan.md + design-doc.md |
| **foundry:forge** | `/forge`, "new task", "forge this" | Full task executor — takes plan + design → committed, tested code via coordinated agents |

```
Crucible (planning)  →  Forge (execution)
─────────────────       ─────────────────
Task description        plan.md
Architecture doc   →    design-doc.md
Codebase context        architecture doc
                        ↓
                        Committed, tested code
```

## Crucible — Multi-Model Plan Refinery

Takes a raw task and refines it through 3 AI models (Opus, Codex, Gemini) using the **RALPH Loop** — an adaptive convergence pattern where models cross-review each other until they agree.

### Phases
| Phase | Name | Description |
|-------|------|-------------|
| 1 | **Intake** | Resume check → collect task, optional docs, output dir |
| 2 | **Context** | Read inputs, detect platform, assemble shared context packet |
| 3 | **Fleet Dispatch** | 3 models independently draft plan + design doc |
| 4 | **Convergence** | RALPH loop — cross-review until all 3 agree (max 10 rounds) |
| 5 | **Validate** | Merge, validate against Forge's required fields |
| 6 | **Output** | Write final files, cleanup intermediates |
| 7 | **Actions** | Hand off to Forge / push to GitHub / done |

### Output
- `plan.md` — Splits, file breakdowns, dependencies (DAG), acceptance criteria
- `design-doc.md` — Problem, solution, types, APIs, error handling, alternatives

## Forge — Full Task Executor

Takes Crucible's output (or user-provided docs) and executes the plan through coordinated agents with build verification and deep review.

### Phases
| Phase | Name | Description |
|-------|------|-------------|
| 1 | **Gather** | Resume check → collect document locations |
| 2 | **Locate** | Find and read all documents |
| 3 | **Analyze** | Extract requirements, validate plan, generate summaries |
| 4 | **Readiness** | Structured report of gaps and warnings |
| 5 | **Confirm** | Final execution preview with user approval |
| 6 | **Execute** | RALPH loop — per-file code agents + verifiers per split |
| 7 | **Review** | Deep review with adversarial multi-perspective analysis |

### The RALPH Loop

Named after the orchestrator pattern: allocate resources with backing specs, give them a goal, loop until done.

> "Software is clay on the pottery wheel. If something isn't right, throw it back on the wheel."

- **In Crucible**: Models cross-review plans until convergence
- **In Forge**: Code agents iterate with verifier feedback until all checks pass

## Agents

### Crucible Agents
| Agent | Purpose |
|-------|---------|
| `plan-drafter` | Generates/revises Forge-compatible plan.md |
| `design-drafter` | Generates/revises Forge-compatible design-doc.md |

### Forge Agents
| Agent | Purpose |
|-------|---------|
| `code-agent` | Per-file code implementation |
| `plan-verifier` | Checks code against plan (every iteration) |
| `design-verifier` | Checks code against design doc (at split completion) |
| `architecture-verifier` | Checks code against architecture (at split completion) |
| `scribe` | Conditional task logging |

## Models Used (Crucible)

| Model | ID | Strength |
|-------|-----|----------|
| Claude Opus | `claude-opus-4.6` | Deep reasoning, architecture |
| GPT Codex | `gpt-5.1-codex-max` | Code-first planning |
| Gemini Pro | `gemini-3-pro-preview` | Breadth, alternatives |

## Safety Features

- **Adaptive convergence** (Crucible) — never forces premature consensus
- **Hard caps** — 10 rounds in Crucible, 10 iterations per split in Forge
- **Resume support** — both skills track state and can pick up where they left off
- **Build gates** (Forge) — auto-detect build system, test before every commit
- **Git safety** (Forge) — preflight checks, fetch/rebase before push, specific file staging
- **Clean intermediates** — working files deleted, only outputs survive

## Installation

```bash
copilot plugin install shshivakumar_microsoft/foundry
```

## Version

- **v1.0.0** — Initial release combining Crucible + Forge under one plugin
