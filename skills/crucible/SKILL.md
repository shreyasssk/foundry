---
name: crucible
description: Multi-model plan refinery — dispatches tasks to 3 AI models (Opus, Codex, Gemini) in parallel, cross-reviews adaptively until convergence, and outputs Forge-compatible plan.md + design-doc.md. Triggers on "plan this", "crucible", "/crucible", "design this task", "create a plan for", "I need a plan", "refine this plan".
disable-model-invocation: true
---

# Crucible — Multi-Model Plan Refinery

> **Named after the metallurgy concept:** raw materials are refined in a crucible before entering the forge.

Crucible takes a raw task description, dispatches it to 3 AI models in parallel, cross-pollinates their outputs through adaptive convergence rounds, and produces Forge-compatible `plan.md` + optionally `design-doc.md`.

**Not all tasks need a design doc.** Crucible assesses task complexity and recommends whether a design doc is needed. Small, straightforward tasks skip design doc generation entirely.

---

## Phase 1 — Task Intake & Resume Detection

### Resume Check

Before asking for input, check if `crucible-state.md` exists in the Crucible working directory:

1. Search for `crucible-state.md` in `~/.copilot/foundry/*/` (scan all task directories)
2. If found, read and validate:
   - Is the round counter consistent?
   - Are intermediate files still present?
   - Is the task description still available?
3. If valid, present to user:
   ```
   Found existing Crucible state:
     Task      : <task summary>
     Round     : <N> / 10
     Models    : <list>
     Status    : <converging/paused/failed>
     
   Options:
   1. Resume from round <N>
   2. Start fresh (will archive old state)
   ```
4. If resume chosen, skip to Phase 4 at the indicated round.
5. If state invalid, warn and recommend fresh start.

### New Task Intake

If no state found or fresh start chosen, ask for:

```
To refine a plan with Crucible, provide:

1. Task description  — text, file path, URL, or work item ID
2. Architecture doc  — (optional) global/shared constraints
3. Existing plan     — (optional) if you already have a plan.md, provide it for validation
```

> **Design doc question is complexity-gated — only prompted for medium+ tasks.** Do NOT ask about design docs during initial intake. After collecting items 1-3, proceed to § Complexity Assessment. If complexity is assessed as `large`, THEN ask: "Do you want a design document? (Recommended for complex tasks.)" If complexity is `small`, skip the design doc question entirely — no mention, no prompt. This prevents overwhelming users with unnecessary choices on simple tasks.

All Foundry working files and outputs live **outside the repo** at `~/.copilot/foundry/<task-slug>/`. Nothing is written to the source directory. Forge reuses the same directory.

After user responds, ask for **execution configuration** — Forge reads these from the plan and runs headless:

```
Forge execution config (so Forge can run without prompting):

5. Base branch       — which branch should Forge create splits from?
                       (e.g., main, master, develop, build/main/latest)
6. Branch prefix     — what naming convention for split branches?
                       Common patterns:
                         a. user/<alias>/<task-name>
                         b. feature/<task-name>
                         c. forge/<task-name>
                         d. Custom prefix
                       (default: forge/<task-name>)
7. Split relationship — (ONLY if split-strategy is MULTI)
                        Are splits chained (each builds on previous) or independent?
                        (default: chained)
                        • chained: each split branches from the previous split's branch
                        • independent: each split branches from the BASE branch (not from prior splits), enabling clean parallel merges
                        ⚠️ SKIP this question if split-strategy is single — set to N/A.

> **Important**: This question is conditional — only ask if split strategy (assessed below) results in `multi`. If split strategy is `single`, set `Split Relationship: N/A` and skip this question. Assess split strategy FIRST (see § Split Strategy Assessment), then return here for item #7 if needed.

**Note:** Independent splits are logically independent (each branches from base, enabling independent merges) but execute sequentially in the current implementation — one Forge invocation per split. Parallel execution is a future enhancement.

> **Concurrency note**: `forge-state.md` is designed for single-agent sequential access. Foundry does not support parallel split execution — splits execute sequentially even when independent. The `independent` relationship affects branch topology (rebase target), not execution concurrency. Do not attempt concurrent writes to `forge-state.md` or `crucible-state.md`.
```

**All applicable items must be collected before proceeding.** Items #5-6 are always required. Item #7 is only required when split-strategy is `multi`. Prompt explicitly for anything missing — Forge will not ask again.

After user responds:

- If existing plan provided → enter **Validation Mode** (Phase 1.5)
- If existing design doc provided but no plan → enter **Generation Mode** with design doc as context
- If no design doc → proceed to **Complexity Assessment** below (do NOT ask about design doc yet)
- Detect platform from `git remote -v` (ADO vs GitHub) for work item fetching
- Record ALL choices in memory including execution config
- Store execution config in `crucible-state.md` as `base-branch`, `branch-prefix`, `split-relationship`

### Complexity Assessment

Before deciding on design doc generation, assess the task's complexity based on the task description and any provided context:

**Indicators of a SMALL task** (design doc NOT needed):
- Touches ≤ 5 files
- Single split likely sufficient
- Bug fix, config change, simple feature addition
- No new interfaces, APIs, or data model changes
- No cross-component or cross-service interaction
- Clear, well-scoped change with obvious approach

**Indicators of a LARGE task** (design doc recommended):
- Touches > 5 files or multiple components
- Multiple splits needed
- New APIs, interfaces, services, or data models
- Cross-component interactions or new integration points
- Architectural decisions required
- Multiple valid approaches exist
- Error handling or security implications

After assessment, present your recommendation to the user:

```
## Complexity Assessment

Based on the task description, this appears to be a [SMALL / LARGE] task.

[2-3 sentences explaining why — e.g., "This is a localized bug fix touching 2 files
with no new interfaces or API changes."]

Recommendation:
  ☐ Skip design doc — proceed with plan only (faster, less overhead)
  ☐ Dispatch design-drafter agent alongside plan (recommended for complex tasks)

Your choice?
```

Record the decision. This sets the `complexity` field in the output plan:
- **SMALL** → `complexity: small` in plan.md header, no design doc generated, Forge skips design doc and architecture doc requirements
- **LARGE** → `complexity: large` in plan.md header, design doc generated, Forge requires full ceremony

Store in `crucible-state.md` as `complexity: small|large` and `generate-design: true|false`.

### Split Strategy Assessment

After complexity is decided, assess whether the task requires multiple splits or can be completed in a single branch:

**Indicators of SINGLE branch** (no splits needed):
- SMALL task that touches ≤ 5 files
- All changes are tightly coupled — no logical separation point
- No meaningful dependency ordering between files
- Entire change can be reviewed as one coherent unit

**Grey zone (6–7 files):**
- Not clearly single or multi — present both options with pros/cons
- Lean towards single if changes are tightly coupled; lean towards multi if there are natural phase boundaries
- Always let the user decide for this range

**Indicators of MULTI split** (splits recommended):
- LARGE task with distinct logical phases
- Changes span multiple components that can be developed independently
- Clear dependency ordering (e.g., "data model first, then API, then UI")
- Total file count > 8 — splitting aids reviewability

Present the recommendation:

```
## Split Strategy

Based on the scope, this task [can be completed in a single branch / should be split into N parts].

[1-2 sentences explaining why — e.g., "All 3 files are tightly coupled and form a single
logical change. Splitting would add overhead without improving reviewability."]

Recommendation:
  ☐ Single branch — all changes in one branch, no split overhead
  ☐ Multiple splits — [N] splits as outlined above (recommended for larger tasks)

Your choice?
```

Record the decision:
- **SINGLE** → `split-strategy: single` in plan.md's `## Execution Config`, plan still has a `## Splits` section but with exactly ONE split containing all files
- **MULTI** → `split-strategy: multi` in plan.md's `## Execution Config`, plan has multiple splits as normal

Store in `crucible-state.md` as `split-strategy: single|multi`.

**Important:** If the user picks SINGLE, the execution config questions about split relationship (#7) become irrelevant — skip asking about chained vs independent and set `Split Relationship: N/A`.

---

## Phase 1.5 — Validation Mode (Existing Documents)

If the user provides an existing `plan.md` and/or `design-doc.md`, Crucible validates them against Forge's requirements **before** deciding whether to refine.

### Plan Validation

Read the provided plan and check every Forge requirement:

| # | Requirement | Check |
|---|------------|-------|
| 1 | Task splits present | Are there named splits with clear goals? |
| 2 | Splits meaningfully scoped | Is each split independently testable? Not too large (>12 files)? |
| 3 | Branch name specified | Is there a `## Execution Config` section with a valid git branch name? |
| 4 | File-level breakdown per split | Does every split have a files table with exact paths? |
| 5 | File actions specified | Does every file have CREATE or MODIFY action? |
| 6 | File dependencies specified | Are dependencies listed? Are they free of circular references? |
| 7 | Acceptance criteria per split | Are criteria specific and testable (not vague)? |
| 8 | Test strategy per split | Is there a concrete test approach? |
| 9 | Splits ordered correctly | Do later splits only depend on earlier ones? |
| 10 | No contradictions between splits | Do splits reference consistent file paths and interfaces? |
| 11 | Complexity section present | Is there a `## Complexity` section with `Classification: small` or `Classification: large`? If missing, assess the plan's scope and add the section — Forge requires it to determine which verifiers to run. |
| 12 | Execution config present | Is there a `## Execution Config` section with `Base Branch`, `Branch Prefix`, `Split Strategy`, and (if multi) `Split Relationship`? If missing, add it from the user's intake answers — Forge requires it to run headless. |
| 13 | Split strategy consistent | If `Split Strategy: single`, the plan must have exactly ONE split in `## Splits` with all files listed. If `Split Strategy: multi`, multiple splits are expected. |

### Design Doc Validation

If a design doc is also provided, check every Forge requirement:

| # | Requirement | Check |
|---|------------|-------|
| 1 | Problem statement | Clear current-state and desired-state? |
| 2 | Proposed solution | Technical approach described? Fits architecture? |
| 3 | Types & interfaces | Real type definitions with actual signatures (not pseudocode)? |
| 4 | API contracts | Endpoints with method, path, request/response types, status codes? |
| 5 | Error handling strategy | Concrete error types, propagation flow, logging? Not just "handle errors"? |
| 6 | Alternatives considered | At least 2 alternatives with specific, technical rejection reasons? Count alternatives — fail if fewer than 2. Rejection reasons must be concrete (not "it's worse"). |

**Strict checks**: Items 1, 3, 4, 5, 6 are blocking — plan/design CANNOT proceed to Forge without these. Item 2 is a warning (non-blocking but flagged).

**Plan checks severity**: Items 1-10 are blocking. Items 11-13 are **[MISSING]** — if absent, Crucible MUST auto-add them from intake answers rather than rejecting the plan. These are Crucible's responsibility to populate.

### Cross-Check (if both provided)

- Does the plan align with the design doc?
- Do the plan's file paths match the design doc's module definitions?
- Does the plan's scope match the design doc's problem statement?

### Validation Report

Present results to the user:

```
╔══════════════════════════════════════════╗
║       Crucible Validation Report         ║
╠══════════════════════════════════════════╣
║                                          ║
║ plan.md:                                 ║
║   ✅ Task splits present (4 splits)      ║
║   ✅ Branch name specified               ║
║   ❌ Split 2 missing file dependencies   ║
║   ❌ Split 3 acceptance criteria vague    ║
║      → "it works" is not testable        ║
║   ⚠️  Split 4 has 15 files — consider    ║
║      splitting further                   ║
║                                          ║
║ design-doc.md:                           ║
║   ✅ Problem statement clear             ║
║   ✅ Types & interfaces defined          ║
║   ❌ Error handling says "handle errors  ║
║      appropriately" — needs specifics    ║
║   ❌ No alternatives considered section  ║
║                                          ║
║ Cross-check:                             ║
║   ⚠️  Plan references UserCache.ts but   ║
║      design doc doesn't mention caching  ║
║                                          ║
╠══════════════════════════════════════════╣
║ Verdict: NOT FORGE-READY                 ║
║ Issues: 3 critical, 2 warnings          ║
╠══════════════════════════════════════════╣
║ Options:                                 ║
║ 1. Auto-fix — Crucible refines the       ║
║    failing sections via multi-model       ║
║    convergence (keeps passing sections)   ║
║ 2. Manual fix — I'll update and re-run   ║
║ 3. Proceed anyway — hand to Forge as-is  ║
║    (Forge will flag the same issues)      ║
╚══════════════════════════════════════════╝
```

**For each ❌ failure**, explain:
- **What** is missing or wrong (the specific requirement)
- **Why** Forge needs it (what breaks without it — e.g., "without file dependencies, Forge spawns agents in random order which may cause import errors")
- **Example** of what good looks like (show the expected format)

### Auto-Fix Path

If user chooses auto-fix:
1. Keep all **passing sections** as-is — do not regenerate what already works
2. Extract only the **failing sections** as the task for the multi-model fleet
3. Proceed to Phase 2 with the context packet focused on fixing specific gaps
4. In Phase 3, instruct models: "Here is an existing plan. These specific sections need fixing: [list]. Keep everything else unchanged."
5. Convergence loop (Phase 4) runs only on the failing sections
6. In Phase 5, merge fixed sections back into the original document

### All-Pass Path

If all checks pass:

```
╔══════════════════════════════════════════╗
║       Crucible Validation Report         ║
╠══════════════════════════════════════════╣
║ plan.md:       ✅ ALL CHECKS PASSED      ║
║ design-doc.md: ✅ ALL CHECKS PASSED      ║
║ Cross-check:   ✅ ALIGNED                ║
╠══════════════════════════════════════════╣
║ Verdict: FORGE-READY                     ║
║                                          ║
║ Options:                                 ║
║ 1. Hand off to Forge (/forge)            ║
║ 2. Done — files are ready                ║
╚══════════════════════════════════════════╝
```

---

## Phase 2 — Context Gathering

### Foundry Working Directory

All Foundry files (Crucible and Forge) live **outside the repo** in a shared directory per task. Derive a slug from the task name and create:

```powershell
# PowerShell — truncate at last whole word boundary within 50 chars
$slug = ($taskName -replace '[^a-zA-Z0-9_-]', '-' -replace '-+', '-').ToLower().Trim('-')
if ($slug.Length -gt 50) {
    $truncated = $slug.Substring(0, 50)
    $lastDash = $truncated.LastIndexOf('-')
    if ($lastDash -gt 0) { $truncated = $truncated.Substring(0, $lastDash) }
    $slug = $truncated
}
$slug = $slug.TrimEnd('-')
$FOUNDRY_DIR = Join-Path $HOME ".copilot" "foundry" $slug
New-Item -ItemType Directory -Path $FOUNDRY_DIR -Force | Out-Null
```
```bash
# Bash — truncate at last whole word boundary within 50 chars
slug=$(echo "$taskName" | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9_-]/-/g' | tr -s '-' | sed 's/^-//; s/-$//')
if [ ${#slug} -gt 50 ]; then
    slug="${slug:0:50}"
    trimmed="${slug%-*}"  # truncate at last dash (word boundary)
    [ -n "$trimmed" ] && slug="$trimmed"  # guard: keep full slug if no dash found
fi
slug="${slug%-}"
FOUNDRY_DIR="$HOME/.copilot/foundry/$slug"
mkdir -p "$FOUNDRY_DIR"
```

> **Word-boundary truncation**: The slug is truncated at the last whole-word boundary (dash) that fits within 50 characters, not mid-word. Example: `"implement-user-authentication-for-sharepoint-online-tenant-admin"` → truncated at 50 chars would be `"implement-user-authentication-for-sharepoint-onlin"` (mid-word), but word-boundary truncation yields `"implement-user-authentication-for-sharepoint"` instead.

_(Full algorithm specification: see § Task Slug Algorithm in ARCHITECTURE.md)_

Store `$FOUNDRY_DIR` in memory. All `crucible-*` files, `plan.md`, and `design-doc.md` go here, NOT in the repo. Forge will later use the same `$FOUNDRY_DIR` for its working files — both skills share one directory per task.

> ⚠️ **Zero repo pollution**: NEVER write `plan.md`, `design-doc.md`, `crucible-state.md`, or ANY Foundry working file to the repository working directory or CWD. ALL Foundry files go to `$FOUNDRY_DIR` (`~/.copilot/foundry/<slug>/`) — no exceptions. Sub-agents (plan-drafter, design-drafter) return content as text; the orchestrator writes it to `$FOUNDRY_DIR`.

### Read Inputs

For each input provided:

- If file path → read directly
- If directory → search for `*.md`, `*.txt`, `architecture*`, `design*`, `plan*`, `spec*`
- If URL → fetch content
- If work item ID → fetch via ADO MCP tools (if ADO detected) or GitHub issues
- If plain text → use directly

Read existing codebase structure:

- Discover project files using cross-platform commands:
  - **Preferred**: Use `git ls-files` to list tracked files (respects `.gitignore`, no limit): `git ls-files -- '*.ts' '*.cs' '*.py' '*.go' '*.java' '*.rs'`
  - **Windows fallback (PowerShell)**: `Get-ChildItem -Recurse -Include *.ts,*.cs,*.py,*.go,*.java,*.rs | Select-Object -First 500 FullName`
  - **Unix fallback**: `find . -type f \( -name "*.ts" -o -name "*.cs" -o -name "*.py" -o -name "*.go" -o -name "*.java" -o -name "*.rs" \) | head -500`
  - Or use the `glob` tool if available: `**/*.{ts,cs,py,go,java,rs}`
  - For very large repos (>500 files), focus on the directories most relevant to the task rather than the full tree
- Read key config files (`package.json`, `tsconfig.json`, `.csproj`, `Cargo.toml`, `go.mod`, etc.)
- Understand the project structure and tech stack

Assemble the **context packet** — a single markdown document that all 3 models will receive identically:

```markdown
# Crucible Context Packet

## Task Description
[full task text]

## Complexity
[small | large]

## Architecture Context
[architecture doc content or "Not provided"]

## Existing Design Doc
[design doc content or "Not provided — Crucible will generate" or "Not needed — small task"]

## Codebase Structure
[file tree summary, tech stack, key patterns observed]

## Instructions
You are one of 3 models participating in a Crucible refinement process.
Your job is to produce a Forge-compatible plan (and design doc if complexity is "large").
Be specific — include exact file paths, concrete types, real method signatures.
Do not be generic or hand-wavy.
**IMPORTANT**: Return your output as text in your response. Do NOT use create/edit/write tools to produce files — the Crucible orchestrator handles all file I/O and writes outputs to the Foundry task directory. You produce content; the orchestrator writes it.
The plan.md MUST include a `## Complexity` section with `Classification: small` or `Classification: large` (see plan-drafter template for full format).
The plan.md MUST include a `## Execution Config` section with the user's branch and split preferences — Forge reads this to run headless. Copy the values from the execution config below.

## Execution Config
Base Branch: [user's chosen base branch]
Branch Prefix: [user's chosen prefix pattern]
Split Strategy: [single | multi]
Split Relationship: [chained | independent | N/A]  ← REQUIRED when Split Strategy is multi; set to N/A when single
```

---

## Phase 3 — Fleet Dispatch (Round 1: Independent Planning)

Dispatch 3 parallel sub-agents using the task tool:

- **Agent 1**: model `claude-opus-4.6` — `mode="background"`
- **Agent 2**: model `gpt-5.1-codex-max` — `mode="background"`
- **Agent 3**: model `gemini-3-pro-preview` — **`mode="sync"` ONLY** (Gemini fails on background dispatch; always use `mode="sync"`, never `mode="background"`)

> ℹ️ Agents 1 and 2 can run in parallel as background agents. Agent 3 (Gemini) MUST be dispatched with `mode="sync"` (not `mode="background"`). A practical pattern: dispatch Opus and Codex as background, then dispatch Gemini synchronously, then collect all results.

Each model dispatches agents from the foundry plugin:

1. **Plan Drafter** — always: `task(agent_type="foundry/plan-drafter", prompt=<context packet + round instructions>, model=<model>)`
2. **Design Drafter** — only if complexity is LARGE and design doc generation is needed: `task(agent_type="foundry/design-drafter", prompt=<context packet + round instructions>, model=<model>)`

If complexity is SMALL, skip the design drafter entirely. Log: `Design drafter skipped — small task (no design doc needed).`

The agent instructions in `agents/plan-drafter.md` and `agents/design-drafter.md` define the exact output templates and quality requirements. Do NOT duplicate those templates here — the agents are the single source of truth.

For each model (Opus, Codex, Gemini), dispatch both agents with the context packet as the prompt. The context packet should include:
- The full context assembled in Phase 2
- Round number (1 for initial)
- Whether design doc generation is needed

Collect outputs from all agents (3 plan-drafters + up to 3 design-drafters if complexity is large).

**Model failure handling**: If a model fails to respond (timeout, API error, empty output):
1. Retry once with the same model after 10 seconds
2. If retry fails, use `claude-sonnet-4.6` as a universal fallback model
3. If fallback also fails, proceed with 2-model convergence — log the gap:
   ```
   ⚠️ [model] failed (retry + fallback exhausted). Proceeding with 2-model convergence.
   ```
   With 2 models, convergence requires BOTH models to agree on all structural checks (since ≥2 of 2 is trivially true, explicit unanimous agreement is required).
4. If 2+ models fail, STOP and ask the user — 1-model output is not a valid convergence.

Store each model's output in `$FOUNDRY_DIR`:

- `crucible-round-1-opus.md` (plan-drafter output)
- `crucible-round-1-codex.md` (plan-drafter output)
- `crucible-round-1-gemini.md` (plan-drafter output)
- If complexity is `large` (design doc needed), also store:
  - `crucible-round-1-opus-design.md` (design-drafter output)
  - `crucible-round-1-codex-design.md` (design-drafter output)
  - `crucible-round-1-gemini-design.md` (design-drafter output)

Write initial `crucible-state.md` to `$FOUNDRY_DIR`:

```markdown
# Crucible State
Task      : [summary]
Slug      : [task-slug]
Round     : 1 / 10
Status    : round-1-complete
Models    : [opus, codex, gemini]
Complexity: [small | large]
Generate  : [plan-only | plan-and-design]
Split-Strategy: [single | multi]
Split-Relationship: [chained | independent | N/A]
Base-Branch   : [branch name]
Branch-Prefix : [from intake, default: foundry/]

## Round History
| Round | Opus | Codex | Gemini | Converged |
|-------|------|-------|--------|-----------|
| 1     | done | done  | done   | pending   |
```

---

## Phase 4 — Cross-Review Loop (Adaptive Convergence)

This is the **RALPH Loop** — named after the orchestrator pattern where you allocate resources with backing specifications, give them a goal, and loop until the goal is met.

The philosophy: "Software is clay on the pottery wheel. If something isn't right, throw it back on the wheel."

**For each round (2..N, max 10):**

1. Dispatch 3 parallel agents (same models as Round 1)
2. Each agent receives:
   - The original context packet
   - ALL outputs from the PREVIOUS round (all 3 models' outputs)
   - Round number and instruction:

```
You are participating in a Crucible refinement process (Round [N] of max 10).

You previously produced a plan (and possibly design doc). You are now reviewing
the outputs from ALL 3 models (including your own from last round) and producing
a REVISED version.

## Previous Round Outputs
[paste all 3 models' outputs from round N-1]

## Your Task
1. Read all 3 outputs carefully
2. Identify the BEST ideas from each
3. Identify disagreements, gaps, or weaknesses
4. Produce your REVISED plan.md (and design-doc.md if applicable) — return as text, do NOT write files
5. At the END, add a convergence assessment:

### Convergence Assessment
- CONVERGED: [yes/no]
- If no, list remaining disagreements:
  1. [specific disagreement]
  2. [specific disagreement]
- Key changes from your previous version:
  1. [what you changed and why]
```

3. Store outputs to `$FOUNDRY_DIR`: `crucible-round-N-opus.md`, `crucible-round-N-codex.md`, `crucible-round-N-gemini.md`. All intermediate and final files go to `$FOUNDRY_DIR` — never to the repository working directory.

4. **Convergence Detection**: After all 3 complete, check:
   - Did all 3 say "CONVERGED: yes"? → **Exit loop, proceed to Phase 5**
   - Did 2/3 say converged? → Run ONE more round for the holdout
   - Did none converge? → Continue to next round

### Structural Convergence Verification

Do NOT rely solely on models self-reporting "CONVERGED: yes". After collecting round N outputs, the orchestrator performs independent structural checks:

**Plan convergence checks** (always apply):
1. **Branch name** — exact match across all 3 models
2. **Split count** — same number of splits
3. **Split names** — all 3 models describe the same work in each split (same intent, same scope — wording differences are fine)
4. **File lists per split** — all 3 models list the same files for each split (minor differences of 1-2 files are acceptable if the extra files are reasonable)
5. **Dependency graph** — all 3 models agree on which files depend on which other files (same edges, same direction — ordering of independent files may differ)

**Design doc convergence checks** (large tasks only — skip if complexity is small):
6. **Solution approach** — all 3 models agree on the technical approach
7. **Key types/interfaces** — all 3 models define the same types with compatible signatures
8. **API contracts** — all 3 models agree on endpoints, methods, and status codes

If complexity is small (plan-only mode), only checks 1-5 apply. Do not attempt to compare design doc outputs that don't exist.

**Convergence criteria** (ALL must be true):
- ≥ 2 of 3 models self-report CONVERGED: yes
- Plan structural checks 1-5 pass
- Design doc structural checks 6-8 pass (large tasks only)
- No model introduced NEW splits or removed existing ones from round N-1

If models say converged but structural checks fail → override: NOT converged, continue looping.
If structural checks pass but models say not converged → accept convergence (models are being overly cautious).

5. Update `crucible-state.md` with round results

6. **Hard cap at round 10**: If not converged after 10 rounds, apply the **Divergent Merge Strategy** before presenting options:

   **Divergent Merge Strategy** — when cap is reached without convergence, use this priority:
   1. If the disagreement is **architectural**, use the Opus/Sonnet output as base.
   2. If the disagreement is **implementation-focused**, use the Codex output as base.
   3. For **mixed disagreements**, use the output that satisfies the most structural convergence checks.
   
   Always note in the plan header: `convergence: partial (cap-reached)`.

```
Crucible has not fully converged after 10 rounds.
Remaining disagreements:
[list each specific point where models still disagree]

Divergent merge applied: [which model's output was used as base and why]

Options:
1. Accept best-effort merged output (recommended)
2. Pick one model's output as winner
3. Extend by 5 more rounds
4. Abort
```

**IMPORTANT for Gemini**: Always dispatch Gemini with `mode="sync"` — it consistently fails on `mode="background"` dispatch. Opus and Codex can use background mode.

---

## Phase 5 — Merge & Validate

Take the converged outputs and produce final unified `plan.md` and `design-doc.md`.

### Merging Strategy

- If all 3 produced nearly identical output → use any one as base, incorporate unique good ideas from others
- If 2/3 agreed → use the majority as base
- If still some divergence → take the most detailed/specific version as base

### plan.md Validation (Forge Requirements)

Run these checks — ALL must pass:

- [ ] Task splits present and meaningfully scoped
- [ ] Branch name specified
- [ ] File-level breakdown per split with exact paths
- [ ] File dependencies specified and free of circular references — validate dependency DAG using topological sort (Kahn's algorithm), consistent with Forge Phase 3 validation
- [ ] Acceptance criteria per split (specific, testable)
- [ ] Test strategy per split
- [ ] No contradictions between splits
- [ ] Splits are ordered respecting inter-split dependencies
- [ ] Split dependencies form a valid DAG (no cycles). If a cycle is detected, flag it in the readiness report

#### DAG Re-Validation After Plan-Drafter

After plan-drafter generates the split ordering, Crucible validates the dependency DAG:
1. **No cycles** — verify no cycles exist via topological sort (Kahn's algorithm). If a cycle is found, report the exact cycle path (e.g., "Split 2 → Split 3 → Split 2").
2. **All references valid** — confirm all referenced split numbers exist in the plan. Flag any `depends_on` entry pointing to a non-existent split.
3. **Order respects dependencies** — confirm the execution order respects all declared dependencies (no split executes before its prerequisites).

If validation fails, re-dispatch plan-drafter with the specific error:
```
DAG validation failed:
  [specific error, e.g., "Cycle detected: Split 2 → Split 3 → Split 2" or "Split 4 depends on non-existent Split 7"]
Fix the dependency graph and re-emit the plan.
```

If any other check fails → dispatch one targeted model round to fix ONLY the failing checks.

### design-doc.md Validation (Forge Requirements)

**Skip this section entirely if complexity is small (plan-only mode).** Design doc validation only applies when a design doc was generated.

Run these checks — ALL must pass:

- [ ] Problem statement present and clear
- [ ] Proposed solution approach described
- [ ] New types/interfaces/classes listed with signatures
- [ ] API contracts and data model changes specified
- [ ] Error handling strategy defined (concrete, not vague)
- [ ] Alternatives considered with rejection reasons

If any check fails → dispatch one targeted model round to fix ONLY the failing checks.

### Cross-Check (same as Forge Phase 3 — large tasks only)

Skip cross-check if complexity is small (no design doc to cross-check against).

- Does the plan align with the design doc?
- Does the design respect architecture constraints (if arch doc provided)?
- Are there contradictions between plan and design?
- Are file dependencies free of circular references across ALL splits?

---

## Phase 6 — Output & Cleanup

Write final files to `$FOUNDRY_DIR`:

```
~/.copilot/foundry/<task-slug>/
  plan.md               ← always
  design-doc.md         ← only if complexity is large and design doc was generated
```

### Safe Cleanup (two-phase)

**Rule 8 gate**: Do not delete ANY files until the user has chosen an option (Hand off / Done) from the completion summary below.

1. **Write final files first** — write `plan.md` (always) and `design-doc.md` (large tasks only) to `$FOUNDRY_DIR`
2. **Verify outputs exist** — confirm `plan.md` exists and is non-empty. For large tasks, also confirm `design-doc.md` exists and is non-empty.
3. If verification fails — DO NOT delete intermediates. Warn the user:
   ```
   ⚠️ Output verification failed. Intermediate files preserved for recovery.
   ```
4. **Present completion summary** (see below) and **wait for user choice**.
5. **THEN delete intermediates** — only after user chooses "Hand off" or "Done":
   - `crucible-round-*.md` (all round outputs)
   - `crucible-state.md` (convergence tracker)

Present completion summary:

```
╔══════════════════════════════════════╗
║         Crucible Complete            ║
╠══════════════════════════════════════╣
║ Rounds     : [N] (converged at [N]) ║
║ Models     : Opus, Codex, Gemini    ║
║ Complexity : [small | large]        ║
║                                      ║
║ Output:                              ║
║   → plan.md       [X splits, Y files]║
║   → design-doc.md [generated/skipped]║
║                                      ║
║ Location: $FOUNDRY_DIR              ║
╠══════════════════════════════════════╣
║ Options:                             ║
║ 1. Hand off to Forge (/forge)        ║
║ 2. Done — review manually            ║
╚══════════════════════════════════════╝
```

---

## Phase 7 — Optional Actions

### Option 1: Hand off to Forge

- Tell the user to run `/forge` and point it at `$FOUNDRY_DIR/plan.md`
- Forge will pick up `plan.md` automatically (and `design-doc.md` if present for large tasks)
- Forge uses the same `$FOUNDRY_DIR` — no need to specify a separate output path
- Example: `Forge this. Plan is at ~/.copilot/foundry/<task-slug>/plan.md`

### Option 2: Done

- Confirm files are written to `$FOUNDRY_DIR` and exit

---

## Rules

1. **Never skip phases** — even if the task seems simple
2. **Never fabricate convergence** — if models disagree, keep looping (up to cap)
3. **Identical context packets** — all 3 models must receive byte-identical input each round
4. **Forge compatibility is non-negotiable** — `plan.md` and `design-doc.md` MUST pass Forge's Phase 3 parser
5. **Platform awareness** — detect ADO vs GitHub from git remote, use appropriate MCP tools
6. **Clean up intermediates** — only `plan.md` (and `design-doc.md` for large tasks) survive in `$FOUNDRY_DIR`
7. **Gemini sync mode** — always dispatch Gemini with `mode="sync"`, never background
8. **User approval before cleanup** — don't delete intermediate files until the user has chosen an option in Phase 6
9. **State persistence** — write `crucible-state.md` to `$FOUNDRY_DIR` after every round so the process can resume
10. **Delimiter parsing** — when parsing `crucible-state.md` or any coordination file that uses `---` as section delimiters, only count `---` delimiters that appear at column 0 outside of fenced code blocks (``` regions). Delimiters inside code fences are content, not structure. Match `^---$` on its own line. Exclude lines inside fenced code blocks (track fence open/close state by counting triple-backtick lines). Ignore `---` inside HTML comments.
11. **Token cost awareness** — in rounds 2+, only pass the PREVIOUS round's outputs (not all historical rounds) to keep context manageable
12. **Zero repo pollution** — never write any Crucible files to the source repo. Only Forge writes code changes to the repo.
