# Foundry — Complete AI Task Lifecycle

> The foundry contains both the crucible (where plans are refined) and the forge (where code is shaped).

## Two Skills, One Workflow

| Skill | Trigger | What It Does |
|-------|---------|-------------|
| **foundry:crucible** | `/crucible`, "plan this", "design this task" | Multi-model plan refinery — 3 AI models converge on a Forge-compatible plan.md + design-doc.md |
| **foundry:forge** | `/forge`, "new task", "forge this" | Full task executor — takes plan + design → committed, pushed code via coordinated agents |

```
Crucible (planning)  →  Forge (execution)
─────────────────       ─────────────────
Task description        plan.md
Architecture doc   →    design-doc.md
Codebase context        architecture doc (required)
                        ↓
                        Committed & pushed code
                        (user creates PR)
```

## Crucible — Multi-Model Plan Refinery

Takes a raw task and refines it through 3 AI models (Opus, Codex, Gemini) using the **RALPH Loop** — an adaptive convergence pattern where models cross-review each other until they agree. Convergence checks use plain-language descriptions to determine agreement.

### Phases
| Phase | Name | Description |
|-------|------|-------------|
| 1 | **Intake** | Resume check → collect task, docs, output dir |
| 2 | **Context** | Read inputs, assemble shared context packet |
| 3 | **Fleet Dispatch** | 3 models independently draft plan + design doc via `task(agent_type="foundry/plan-drafter")` and `task(agent_type="foundry/design-drafter")` |
| 4 | **Convergence** | RALPH loop — cross-review with plain-language convergence checks until all 3 agree (max 10 rounds) |
| 5 | **Validate** | Merge, validate against Forge's required fields |
| 6 | **Output** | Write final files, cleanup intermediates |
| 7 | **Actions** | Hand off to Forge or done |

### Output
- `plan.md` — Splits, file breakdowns, dependencies (DAG), acceptance criteria
- `design-doc.md` — Problem, solution, types, APIs, error handling, alternatives

## Forge — Full Task Executor

Takes Crucible's output (or user-provided docs) and executes the plan through coordinated agents. Pushes committed code to the remote branch — PR creation is the user's responsibility. Architecture doc is required.

### Phases
| Phase | Name | Description |
|-------|------|-------------|
| 1 | **Gather** | Resume check → collect document locations |
| 2 | **Locate** | Find and read all documents (architecture doc required) |
| 3 | **Analyze** | Extract requirements, validate plan, generate summaries |
| 4 | **Readiness** | Structured report of gaps and warnings |
| 5 | **Confirm** | Final execution preview with user approval |
| 6 | **Execute** | RALPH loop — per-file code agents + verifiers per split |
| 7 | **Build** | Full build gate — runs once after all splits complete |
| 8 | **Review** | Local deep review against branch diff with adversarial multi-perspective analysis |

### The RALPH Loop

Named after the orchestrator pattern: allocate resources with backing specs, give them a goal, loop until done.

> "Software is clay on the pottery wheel. If something isn't right, throw it back on the wheel."

- **In Crucible**: Models cross-review plans until plain-language convergence
- **In Forge**: Code agents iterate with verifier feedback until all checks pass

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
| `design-verifier` | `task(agent_type="foundry/design-verifier")` | Checks code against design doc (at split completion) |
| `architecture-verifier` | `task(agent_type="foundry/architecture-verifier")` | Checks code against architecture (at split completion) |
| `scribe` | `task(agent_type="foundry/scribe")` | Conditional task logging |

## Models Used (Crucible)

| Model | ID | Strength |
|-------|-----|----------|
| Claude Opus | `claude-opus-4.6` | Deep reasoning, architecture |
| GPT Codex | `gpt-5.1-codex-max` | Code-first planning |
| Gemini Pro | `gemini-3-pro-preview` | Breadth, alternatives |

## Safety Features

- **Adaptive convergence** (Crucible) — plain-language convergence checks, never forces premature consensus
- **Hard caps** — 10 rounds in Crucible, 10 iterations per split in Forge
- **Resume support** — both skills track state and can pick up where they left off
- **Post-execution build gate** (Forge) — full build after all splits complete, before deep review
- **Git safety** (Forge) — fetch/rebase before push, specific file staging, dynamic default branch detection
- **Architecture required** (Forge) — architecture doc is mandatory for execution
- **Local deep review** (Forge) — runs against branch diff, no PR needed
- **Clean intermediates** — working files deleted, only outputs survive

## Installation

```bash
copilot plugin install shshivakumar_microsoft/foundry
```

## Version

- **v1.2.0** — No PR creation (push-only), build gate moved post-execution, local deep review, architecture doc required, explicit agent dispatch, plain-language convergence checks
- **v1.1.0** — Fixed all 10 issues from multi-model evaluation (infinite loop protection, structural convergence, cross-platform, safe cleanup, rollback/retry, and more)
- **v1.0.0** — Initial release combining Crucible + Forge under one plugin
