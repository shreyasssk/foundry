---
name: forge
description: Use this skill for the full task lifecycle — intake, analysis, validation, and coordinated multi-agent execution with build verification and deep review. Triggers on "new task", "here's the task", "work on this", "forge this", "I have a task for you", "/forge", "resume", "continue".
disable-model-invocation: true
---

# Forge

Forge is a full task lifecycle skill. It takes a task from raw documents to finished, committed code through structured intake, validation, build verification, and a coordinated multi-agent execution loop.

**Never touch code before Phase 5 confirmation. Never skip phases.**

---

## Phase 1 — Gather Document Locations (with Resume Detection)

### Resume Check

Before asking for documents, check if a `forge-state.md` already exists in the Forge working directory:

1. Search for `forge-state.md` in `~/.copilot/foundry/*/` (scan all task directories)
2. If multiple state files found, filter by repo root match (compare recorded cwd in state file to current working directory). If multiple remain, present a numbered list sorted by last-modified time and let the user choose.
3. If found (single match or user-selected), read it and validate:
   - Does the branch still exist? (`git branch --list <branch>`)
   - Are the split/iteration counters consistent?
   - Is the working tree clean? (`git status --porcelain`)
   - Does `base-branch` exist? If missing (legacy state from pre-v1.3.5), ask the user to provide it before resuming.
   - Does `complexity` exist? If missing (legacy state from pre-v1.4.0), check plan for `## Complexity` section first. If plan also lacks it, prompt user: `'No complexity classification found — classify as small or large?'` Only default to `large` as absolute last resort if user is unavailable. Log: `⚠️ Pre-v1.4.0 state detected — resolved complexity to <actual> via [plan | user prompt | large fallback].`
   - Does `split-strategy` exist? If missing (pre-v1.5.0 state): first check the plan's `## Execution Config` section for a `Split Strategy` field. If found, use it. If not found in the plan either, log `⚠️ Pre-v1.5.0 state detected — cannot reliably infer split-strategy. Defaulting to multi (full ceremony). Override with user input if incorrect.` and default to `multi`.
    - Does `split-relationship` exist? If `split-strategy` is `multi` and `split-relationship` is missing: log `⚠️ split-relationship missing for multi-split state. Prompting user.` Ask user: `'Split relationship not found — chained or independent?'` Default to `chained` if user is unavailable.
4. If valid, present to the user:
   ```
   Found existing Forge state:
     Branch    : <branch>
     Base      : <base-branch>
     Split     : <N> / <total>
     Iteration : <I>
     Phase     : <phase>
     Status    : <status>

   Options:
   1. Resume from where we left off
   2. Start fresh (archives old state to `forge-state-archive-<timestamp>.md` in the same directory)
   ```
4. If the user chooses to resume, skip to the phase indicated in `forge-state.md`.
5. If state is invalid (branch deleted, inconsistent counters), warn and recommend starting fresh.

### New Task Intake

If no state found or user chose fresh start, use a **two-step intake** to avoid asking for unnecessary documents:

**Step 1 — Get the plan (always required):**
```
To get started with Forge, please provide the path to your plan.
You can give a file path or a directory (I'll find the right files inside).

1. Plan               — REQUIRED (file or folder path)
2. Task description   — brief summary or work item ID
```

Wait for the user's response. Read the plan and check for a `## Complexity` section.

**Step 2 — Check complexity, then conditionally ask for remaining docs:**

- If `Classification: small` is found → this is a Crucible-assessed small task. **Do NOT prompt for design doc or architecture doc.** Display:
  ```
  ⚡ Detected complexity: SMALL (from Crucible plan)
     — Skipping design doc requirement
     — Skipping architecture doc requirement
     Proceeding with plan-only execution.
  ```
  Optionally ask:
  ```
  Any other context? (product spec, additional docs — or press Enter to skip)
  ```

- If `Classification: large` is found (or no complexity section exists) → treat as a large task requiring full ceremony. Ask for remaining docs:
  ```
  Detected complexity: LARGE — requesting full documents.

  Please also provide paths to:
  1. Architecture doc   — REQUIRED for large tasks (file or folder path)
  2. Design doc         — required for large tasks (file or folder path)
  3. Product spec       — optional (skip if not applicable)
  ```

Wait for the user's response. Do not proceed until they reply.

---

## Phase 2 — Locate and Read Documents

For each path provided:
- If it is a directory, search for files matching: architecture*, design*, plan*, spec*, *.md, *.txt
- If it is a file path, read it directly
- If a path resolves to nothing, note it explicitly as not found — do not assume

Read every located file in full. Do not summarize yet — that happens in Phase 3.

---

## Phase 3 — Analyze Documents and Generate Summaries

### Document Analysis

Extract and internalize the following from each document.

**Architecture Doc**
- Components, services, and layers involved
- Key interfaces, data flows, and sequence patterns
- Technical constraints and decisions already locked in
- Security model and trust boundaries

**Design Doc**
- Problem being solved and motivation
- Proposed solution approach
- New types, interfaces, classes, or modules
- API contracts and data model changes
- Error handling strategy
- Alternatives considered and why they were rejected

**Plan**
- Task splits (phases or milestones) — must be meaningful and independently scoped
- Branch name — must be specified
- File-level breakdown per split — which files to create or modify
- File dependency information — which files depend on other files within a split
- Test strategy and acceptance criteria per split

**Product Spec**
- Must-have vs nice-to-have requirements
- Out of scope items
- Success criteria and edge cases

### Cross-Check

- Does the plan align with the design doc?
- Does the design respect architecture constraints?
- Are there contradictions or gaps between any two documents?
- Is the file dependency graph acyclic? Validate DAG using topological sort (Kahn's algorithm): build adjacency list from file dependencies, track in-degree per node, process zero-in-degree nodes iteratively. If any nodes remain after processing, a cycle exists — flag as `[BLOCKING]` with the cycle path and re-enter the RALPH loop to fix the dependency cycle (do not block and wait for the user; attempt an automatic fix first).

### Plan-Specific Validation (required for execution)

- [ ] Complexity section is present in plan
  - If missing → `[BLOCKING]` Plan must include a `## Complexity` section with `Classification: small` or `Classification: large`. Forge cannot determine verifier strategy without it. Offer to classify the plan yourself or ask the user.
- [ ] Architecture doc is provided (LARGE tasks only — skip for small tasks)
  - If missing AND complexity is large → `[BLOCKING]` Architecture doc is required. Provide one or ask Crucible to generate one.
  - If missing AND complexity is small → not blocking, skip silently
- [ ] Design doc is provided (LARGE tasks only — skip for small tasks)
  - If missing AND complexity is large → `[BLOCKING]` Design doc is required for full ceremony. Provide one or ask Crucible to generate one.
  - If missing AND complexity is small → not blocking, skip silently
- [ ] Task splits are present and meaningfully scoped
  - If missing → `[BLOCKING]` — cannot execute without at least one split. Forge requires splits to dispatch code agents.
- [ ] Branch name is specified (in `## Execution Config` → `Branch Prefix` field)
  - If missing → `[MISSING]` warn, offer to define one yourself or ask user
- [ ] File-level breakdown exists per split
  - If missing → `[BLOCKING]` — cannot dispatch per-file code agents without file assignments. Request plan update before proceeding.
- [ ] File dependencies are specified per split
  - If missing → `[BLOCKING]` Forge cannot safely order code agent execution without dependencies. Offer to auto-derive dependencies by analyzing import/include statements in existing files, or ask the user to specify them.
  - If present → validate DAG is acyclic. Cycles are a `[BLOCKING]` error.

### Generate Document Summaries

After analysis, generate condensed summaries for each document. These summaries will be passed to code agents instead of full documents to reduce token cost:

```markdown
# forge-summary-arch.md
[Condensed architecture: key components, layers, boundaries, constraints — max ~500 words]

# forge-summary-design.md
[Condensed design: interfaces, contracts, error handling, data models — max ~500 words]

# forge-summary-plan.md
[Condensed plan: splits overview, file breakdown, dependencies, acceptance criteria — max ~500 words]
```

Write these summaries to `$FOUNDRY_DIR` (`~/.copilot/foundry/<task-slug>/`), not to the repo working directory. Code agents receive summaries; verifiers receive full documents.

---

## Phase 4 — Readiness Report

Present a structured report using this exact format:

```
## Forge Readiness Report

### Documents
- Plan         : [path] — REQUIRED
- Architecture : [path or "SKIPPED — small task"] — required for large tasks
- Design Doc   : [path or "SKIPPED — small task"] — required for large tasks
- Product Spec : [path or NOT PROVIDED / NOT REQUIRED]

### Complexity
[small — plan-only execution | large — full ceremony]
[If small: "Design doc and architecture doc skipped per Crucible complexity assessment"]

### Task Summary
[2-4 sentences: what the task is, proposed approach, scope]

### What Looks Good
- [Each doc found and appears complete]
- [Specific strengths, e.g. "Plan has file-level breakdown with dependencies"]

### Gaps and Warnings
- [BLOCKING]    Complexity section missing from plan — REQUIRED, Forge cannot determine verifier strategy
- [BLOCKING]    Architecture doc not provided — REQUIRED for large tasks, cannot proceed
- [BLOCKING]    Design doc not provided — REQUIRED for large tasks (full ceremony)
- [MISSING]     No branch name in plan
- [MISSING]     No execution config in plan — Forge will prompt for base branch, prefix, split relationship
- [MISSING]     No file-level breakdown in plan — cannot safely spawn per-file agents
- [INCOMPLETE]  Design doc missing error handling strategy
- [BLOCKING]    No file dependencies specified — cannot safely order code agent execution. Offer to auto-derive from imports.
- [MISMATCH]    Plan references module X but design doc does not mention it
- [NOTE]        Product spec marks feature Y as out of scope — confirm before implementing

**For small tasks:** Architecture and design doc gaps are NOT BLOCKING. Do not report them as missing.

### Readiness
READY / NOT READY — [one line reason]
```

If **NOT READY**:
- Ask the user to resolve critical gaps or confirm intentional omissions
- Wait for their response before continuing

If **READY**:
- Proceed to Phase 5

---

## Phase 5 — Final Confirmation (Headless-Ready)

Forge reads all execution config from the plan — **do not prompt the user for branch, prefix, or split strategy.** These are set by Crucible during planning.

### Read Execution Config from Plan

Extract from the plan's `## Execution Config` section:
- `Base Branch` → store as `$BASE_BRANCH`
- `Branch Prefix` → store as `$BRANCH_PREFIX`
- `Split Strategy` → store as `$SPLIT_STRATEGY` (single or multi; default: multi)
- `Split Relationship`:
  - If `$SPLIT_STRATEGY` is `single`: set `$SPLIT_RELATIONSHIP = "N/A"` — the plan-drafter always emits `Split Relationship: N/A` for single-split plans
  - If `$SPLIT_STRATEGY` is `multi`: read `Split Relationship` from `## Execution Config` (required) → store as `$SPLIT_RELATIONSHIP` (chained or independent)

**If `## Execution Config` is missing** (e.g., user-written plan without Crucible):
- Log: `⚠️ No execution config in plan — prompting for required values.`
- Fall back to prompting:
  - Base branch (required)
  - Branch prefix (default: `forge/<task-name>`)
  - Split strategy (default: multi)
  - Split relationship — only ask if split strategy is multi (default: chained)
- This is the ONLY scenario where Forge prompts for these values.

### Validate Execution Config Inputs

Before using any execution config values, validate them against safe patterns.

**Apply this validation to BOTH paths** — values sourced from plan AND values provided by user prompt fallback.

```
Validation rules:
  Base Branch    : must match ^[\w\-\/\.]+$  (alphanumeric, hyphens, slashes, dots only)
                   AND must NOT contain ".." (prevents path traversal)
  Branch Prefix  : must match ^[\w\-\/\.]+$  (same pattern)
                   AND must NOT contain ".." (prevents path traversal)
  Split Strategy : must be exactly "single" or "multi"
  Split Relationship : must be exactly "chained" or "independent"

If any value fails validation:
  - Log: ⚠️ Invalid execution config value for <field>: "<raw-value>" — contains unsafe characters.
  - Do NOT use the value in any git command or shell operation.
  - Fall back to prompting the user for a corrected value.
```

This prevents command injection if a malicious plan.md contains shell metacharacters in branch names.

**If split strategy is `single`**: Log and adapt:
```
ℹ️ Single-branch mode — detected from Crucible plan (split-strategy: single).
  All changes will be committed to a single branch without /split-N suffixes.
```

**If split relationship is `independent`** (only applies when multi): Forge executes ALL splits sequentially in this invocation (one split completes before the next begins). Each split branches from the base branch (not from previous splits). Log and display:
```
ℹ️ Independent split strategy: executing [N] splits sequentially. Each split branches
   from the base branch. Parallel execution across separate Forge processes is a future enhancement.
```

Once all config is resolved (from plan or fallback), show the execution preview:

```
## Ready to Forge

Base     : [base branch]
Branch   : [branch prefix]                          ← if single
           [branch prefix]/split-1..N               ← if multi
Splits   : [1 (single-branch mode) | N splits]
Chaining : [single-branch / chained / independent]
Complexity: [small / large]
Strategy : One code agent per file, parallel within dependency constraints
           Verifiers (plan every iteration; architecture + design at split completion [large tasks] or SKIPPED [small tasks])
           Deep review + build gate [per split (multi) | once (single)] (after verifiers approve, before commit)
           Scribe maintains task log at [project folder path]

Shall I proceed?
```

Wait for explicit user confirmation. Do not touch any file until confirmed. This is the **last prompt before fully headless execution** — after this, Forge runs to completion without interruption (unless a preflight check fails or hard cap is reached).

---

## Phase 6 — Execution

### Setup

Before starting the loop:

#### 1. Environment Preflight Checks

```
# Verify clean working tree (shell-agnostic — works in both PowerShell and Bash)
git status --porcelain
# If output is non-empty → STOP, ask user to stash or commit

# Verify remote is reachable
git ls-remote --exit-code origin
# If fails → STOP, ask user to check network/auth

# Define Write-Utf8NoBom helper (needed for diff generation in Steps 5 and 8)
```
```powershell
function Write-Utf8NoBom($path, $content) {
    if ($PSVersionTable.PSVersion.Major -ge 7) {
        $content | Out-File -Encoding utf8NoBOM $path
    } else {
        [IO.File]::WriteAllText($path, $content, [Text.UTF8Encoding]::new($false))
    }
}
```

If any preflight check fails, do NOT proceed. Report the failure and wait for the user.

**Multi + Independent split guard:** If `split-strategy: multi` and `split-relationship: independent` in forge-state.md, verify that no other split is currently in-progress. Check `forge-state.md → current-split-status`. If another split shows status `in-progress`, STOP — independent splits execute sequentially to avoid forge-state.md conflicts. Tell the user:
```
⚠️ Independent splits must run one at a time. Split <N> is still in-progress.
    Complete or abort split <N> before starting a new one.
```
This prevents concurrent modification of forge-state.md and forge-coordination.md. (Does NOT apply to `split-strategy: single` or chained multi-split.)

#### 2. Foundry Working Directory

All Forge working files live in the shared Foundry directory — the same `$FOUNDRY_DIR` that Crucible uses. If Crucible ran first, the directory (with `plan.md` and `design-doc.md`) already exists.

Derive the slug from the **task name** (same source as Crucible) — extract from the plan's `## Overview` title or the task description provided at intake. Do NOT derive from branch prefix (different input → different slug → broken shared directory contract).

```powershell
# PowerShell — derive slug from task name (must match Crucible's algorithm)
$slug = ($taskName -replace '[^a-zA-Z0-9_-]', '-')
$slug = ($slug -replace '-+', '-').ToLower().Trim('-')
# Truncate at the LAST whole word boundary that fits within 50 characters
if ($slug.Length -gt 50) {
    $truncated = $slug.Substring(0, 50)
    $lastDash = $truncated.LastIndexOf('-')
    if ($lastDash -gt 0) { $slug = $truncated.Substring(0, $lastDash) } else { $slug = $truncated }
}
$slug = $slug.TrimEnd('-')
if ([string]::IsNullOrWhiteSpace($slug)) { $slug = "untitled-task-" + (Get-Date -Format "yyyyMMdd-HHmm") }
# (Full algorithm specification: see § Task Slug Algorithm in ARCHITECTURE.md)
$FOUNDRY_DIR = Join-Path $HOME ".copilot" "foundry" $slug
New-Item -ItemType Directory -Path $FOUNDRY_DIR -Force | Out-Null
```
```bash
# Bash — derive slug from task name (must match Crucible's algorithm)
slug=$(echo "$taskName" | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9_-]/-/g' | tr -s '-' | sed 's/^-//; s/-$//')
# Truncate at the LAST whole word boundary that fits within 50 characters
if [ ${#slug} -gt 50 ]; then
  slug="${slug:0:50}"
  trimmed="${slug%-*}"  # strip back to last hyphen (word boundary)
  [ -n "$trimmed" ] && slug="$trimmed"  # guard: keep full slug if no dash found
fi
slug="${slug%-}"
[ -z "$slug" ] && slug="untitled-task-$(date +%Y%m%d-%H%M)"
# (Full algorithm specification: see § Task Slug Algorithm in ARCHITECTURE.md)
FOUNDRY_DIR="$HOME/.copilot/foundry/$slug"
mkdir -p "$FOUNDRY_DIR"
```

**Where does `$taskName` come from?** If Crucible ran first, read the slug from the existing `crucible-state.md` in `~/.copilot/foundry/*/crucible-state.md` (look for the `Slug:` field). If no Crucible state exists (user-written plan), derive the slug from the plan's title (first `# ` heading) or the `## Overview` section's first sentence — this must match the task name the user originally provided to Crucible, if Crucible was used.

#### 3. Create Coordination Files

Create files in `$FOUNDRY_DIR`:

```
~/.copilot/foundry/<task-slug>/
├── forge-state.md               ← orchestrator loop state
├── forge-coordination.md        ← consolidated verifier feedback
├── forge-verifier-plan.md       ← plan verifier output (isolated)
├── forge-verifier-arch.md       ← architecture verifier output (large tasks only)
├── forge-verifier-design.md     ← design verifier output (large tasks only)
├── forge-task-log.md            ← scribe log (output — survives cleanup)
├── forge-summary.md             ← execution summary (output — survives cleanup)
├── forge-summary-arch.md        ← architecture summary for agents (large tasks only)
├── forge-summary-design.md      ← design summary for agents (large tasks only)
└── forge-summary-plan.md        ← plan summary for agents
```

For **small tasks**, skip creating `forge-verifier-arch.md`, `forge-verifier-design.md`, `forge-summary-arch.md`, and `forge-summary-design.md` — they are not needed.

#### 4. Initialize Task Log

```markdown
# Forge Task Log
Task: [task description]
Task Branch: [task-branch]
Complexity: [small | large]
Split Strategy: [single | multi]
Started: [ISO timestamp]
Splits: [N]
Branching: single branch (<task-branch>)              ← if single
           per-split (<task-branch>/split-1, split-2, ...) ← if multi
---
```

#### 5. Checkout or Create Branch

Branch strategy depends on `$SPLIT_STRATEGY` from the plan's `## Execution Config`:

##### Single-Branch Mode (`$SPLIT_STRATEGY == "single"`)

All changes go on one branch — no `/split-N` suffixes. The branch name IS the task branch prefix directly.

```
ℹ️ Single-branch mode — creating branch: <task-branch>
```

```powershell
# PowerShell — single branch, resume-safe
git fetch origin
$taskBranch = "$BRANCH_PREFIX"
git checkout $taskBranch 2>$null
if ($LASTEXITCODE -ne 0) {
    git checkout -b $taskBranch "origin/$taskBranch" 2>$null
    if ($LASTEXITCODE -ne 0) {
        git checkout -b $taskBranch "origin/$BASE_BRANCH"
    }
}
```
```bash
# Bash — single branch, resume-safe
git fetch origin
git checkout $BRANCH_PREFIX 2>/dev/null \
  || git checkout -b $BRANCH_PREFIX origin/$BRANCH_PREFIX 2>/dev/null \
  || git checkout -b $BRANCH_PREFIX origin/$BASE_BRANCH
```

##### Multi-Split Mode (`$SPLIT_STRATEGY == "multi"`)

Each task split gets its own branch under the same root task, allowing independent review per split. The **source branch** for each split depends on the split relationship:

- **Chained** (`$SPLIT_RELATIONSHIP == "chained"`): Split N branches from split N-1 (builds on previous work).
- **Independent** (`$SPLIT_RELATIONSHIP == "independent"`): Every split branches from `$BASE_BRANCH` directly (no inter-split dependency).

Use the branch prefix and base branch from the plan's `## Execution Config` section (read in Phase 5). The split suffix `/split-N` is always appended automatically. `$BASE_BRANCH` and `$BRANCH_PREFIX` come from the plan.

**Branch naming convention:** `<task-branch>/split-N` (e.g., `user/johndoe/my-feature/split-1`, `feature/my-task/split-2`)

**Important:** The task branch name (`<task-branch>`) must NOT be an existing branch name. Use a path-style name like `user/<alias>/<task-name>` or `feature/<task-name>` — this ensures `<task-branch>/split-N` won't collide with existing refs. If `<task-branch>` exists as a branch, prefix it: `forge/<task-branch>`.

> **Note:** Independent splits execute sequentially (one at a time) despite being logically independent. Each split branches from the base branch rather than the prior split, enabling clean parallel merges. True concurrent execution is a future enhancement.

```bash
git fetch origin
```

**Split 1** — always branches from the user-specified base branch (prefer existing if resuming):
```powershell
# PowerShell — check for existing branch first (resume-safe), then create fresh
$splitBranch = "<task-branch>/split-1"
git checkout $splitBranch 2>$null
if ($LASTEXITCODE -ne 0) {
    git checkout -b $splitBranch "origin/$splitBranch" 2>$null
    if ($LASTEXITCODE -ne 0) {
        # Truly new — create from user-specified base branch
        git checkout -b $splitBranch "origin/$BASE_BRANCH"
    }
}
```
```bash
# Bash — check for existing branch first (resume-safe), then create fresh
git checkout <task-branch>/split-1 2>/dev/null \
  || git checkout -b <task-branch>/split-1 origin/<task-branch>/split-1 2>/dev/null \
  || git checkout -b <task-branch>/split-1 origin/$BASE_BRANCH
```

**Split N (N > 1), Chained** (`$SPLIT_RELATIONSHIP == "chained"`) — branch from the previous split (with full fallback chain):
```powershell
# PowerShell — chained: full fallback chain for resume safety
$splitBranch = "<task-branch>/split-N"
$parentBranch = "<task-branch>/split-<N-1>"
git checkout $splitBranch 2>$null
if ($LASTEXITCODE -ne 0) {
    git checkout -b $splitBranch "origin/$splitBranch" 2>$null
    if ($LASTEXITCODE -ne 0) {
        git checkout -b $splitBranch $parentBranch 2>$null
        if ($LASTEXITCODE -ne 0) {
            git checkout -b $splitBranch "origin/$parentBranch"
        }
    }
}
```
```bash
# Bash — chained: full fallback chain for resume safety
git checkout <task-branch>/split-N 2>/dev/null \
  || git checkout -b <task-branch>/split-N origin/<task-branch>/split-N 2>/dev/null \
  || git checkout -b <task-branch>/split-N <task-branch>/split-<N-1> 2>/dev/null \
  || git checkout -b <task-branch>/split-N origin/<task-branch>/split-<N-1>
```

**Split N (N > 1), Independent** (`$SPLIT_RELATIONSHIP == "independent"`) — branch from the BASE branch, not from prior splits:
```powershell
# PowerShell — independent: branch from base, not prior split
$splitBranch = "<task-branch>/split-N"
git checkout $splitBranch 2>$null
if ($LASTEXITCODE -ne 0) {
    git checkout -b $splitBranch "origin/$splitBranch" 2>$null
    if ($LASTEXITCODE -ne 0) {
        # Independent — always fork from base branch, never from prior split
        git checkout -b $splitBranch "origin/$BASE_BRANCH"
    }
}
```
```bash
# Bash — independent: branch from base, not prior split
git checkout <task-branch>/split-N 2>/dev/null \
  || git checkout -b <task-branch>/split-N origin/<task-branch>/split-N 2>/dev/null \
  || git checkout -b <task-branch>/split-N origin/$BASE_BRANCH
```

Store the current branch name in `forge-state.md` under `current-branch`.

#### 6. Initialize State File

Create `forge-state.md`:

```markdown
---
slug: <task-slug>  ← derived from task name per § Task Slug Algorithm in ARCHITECTURE.md
split: 1
iteration: 1
hard-cap-iterations: 10
deep-review-round: 0
hard-cap-deep-review: 5
status: running
phase: execution
complexity: <small|large>  ← read from plan.md's ## Complexity → Classification field
split-strategy: <single|multi>  ← read from plan.md's ## Execution Config → Split Strategy field
task-branch: <branch-prefix from plan's ## Execution Config>
base-branch: <base-branch from plan's ## Execution Config>
current-branch: <task-branch>              ← if single
                <task-branch>/split-1      ← if multi
split-relationship: <chained|independent from plan's ## Execution Config → Split Relationship>  # set to "N/A" for single-branch mode
---

## Dependency Graph
- FileA.cs: pending
- FileB.cs: pending (depends: FileA.cs)
- FileC.cs: pending

## Agent Status
- FileA.cs: pending
- FileB.cs: blocked
- FileC.cs: pending

## Verifier Status (last iteration)
- Plan        : —
- Architecture: — [or "SKIPPED — small task"]
- Design      : — [or "SKIPPED — small task"]

## Split Tracking
- current-split: 1
- current-split-status: not-started  [not-started | in-progress | complete]

## Deep Review
- Round       : 0
- Last result : —

## Build Gate
- build-fix-attempts: 0

## Failure Log
(none)

## Retry Counts
(tracks consecutive failures per file — mark STUCK at 3)
```

Update `forge-state.md` after every step:
- After code agents complete → update Agent Status
- After dependency round → update Dependency Graph
- After verifiers complete → update Verifier Status
- After failures → append to Failure Log
- On iteration start → increment iteration counter
- On split complete → increment split counter, reset iteration to 1

---

### The RALPH Loop (per split)

> **RALPH** (named after Ralph Wiggum) is an orchestrator pattern for persistent, self-correcting iteration.
> You allocate the resources with the required backing specifications, give it a goal, then loop the goal.
> Each iteration feeds the AI context from its previous work — git history, modified files, verifier feedback —
> creating a self-referential feedback loop that lets the system improve its own output over time.
>
> `forge-state.md` is the RALPH loop's **persistent memory**. It carries the full context of where
> the loop is (split, iteration, dependency status, agent outcomes, verifier verdicts) across iterations.
> Without it, every spin of the wheel starts blind. With it, each iteration is informed by everything
> that came before — failures, corrections, approvals. This is how the loop self-corrects.
>
> The loop runs **per split** — each split is an independent turn on the pottery wheel.
> If something isn't right, it goes back on the wheel. When all verifiers approve,
> the clay is fired (committed). Then the next split goes on the wheel.
>
> Iterations within a split are **sequential** — each iteration completes (code → verify → log)
> before the next begins. Do not start iteration N+1 until iteration N's Scribe entry is written.
>
> In **single-branch mode**, the loop runs exactly once — there is one "split" containing all files.

**Single-branch mode:** If `split-strategy: single` in forge-state.md, the RALPH loop runs for one iteration of the outer loop (one "split" containing all files). No split branching ceremony, no split-N suffixes. The branch was already created in setup. Log at the start:
```
ℹ️ Single-branch mode — processing all files as one unit on branch: <task-branch>
```

**Multi-split mode:** Repeat for each split in order:

**Important:** If the user chose "independent" splits in Phase 5, all splits execute sequentially within this single Forge invocation — each branching from `$BASE_BRANCH` independently. True parallel execution (concurrent Forge instances) is NOT supported. Do NOT launch separate Forge sessions for independent splits. (This check does NOT apply to `split-strategy: single` — single-branch mode always has one unit of work regardless of the `split-relationship` field.)

```
SPLIT START (put the clay on the wheel)
  Create split branch:
    If single-branch mode → already on <task-branch> from setup, skip branch creation
    If multi + split 1 → already on <task-branch>/split-1 from setup
    If multi + split N > 1 (full fallback chain — check existing first, then create):
      Bash: git checkout <split-N> || git checkout -b <split-N> origin/<split-N> || git checkout -b <split-N> <split-N-1> || git checkout -b <split-N> origin/<split-N-1>
      PowerShell: same order — local → origin/self → local-parent → origin/parent
  Update forge-state.md: current-branch = <task-branch> (single) or <task-branch>/split-N (multi)
  Read plan split definition from disk
  Build dependency graph: file → dependencies[]
  Reset agent status map and iteration count
  Write split header to forge-task-log.md

  RALPH LOOP (spin the wheel until the clay is right):
    1. RESOLVE UNBLOCKED FILES
       Files whose dependencies are all in done status are unblocked.
       On first iteration, files with no dependencies are unblocked.

    2. SPAWN CODE AGENTS (parallel, one per unblocked file)

       For each unblocked file, dispatch:
       ```
       task(agent_type="foundry/code-agent", prompt="
         File: <file-path>
         Split: <split-N description>
         Summaries: [forge-summary-plan.md always; forge-summary-arch.md + forge-summary-design.md only if large task]
         Feedback: [latest section of forge-coordination.md, or 'First iteration — no prior feedback']
         Working directory: (located at `<actual $FOUNDRY_DIR path>`)
         Instructions: Implement only what is needed for this file in this split. Do not commit. Do not modify any other file.
       ")
       ```

       _(Note: Gemini dispatch constraints apply only to Crucible fleet dispatch, not to code agents — see ARCHITECTURE.md.)_

       Wait for ALL code agents to complete before continuing. This is a hard synchronization barrier — do NOT proceed to step 3 until every dispatched agent has returned (success or failure). Verify each agent's status explicitly.

    3. HANDLE CODE AGENT FAILURES
       For each failed agent:
         - Retry up to 2 times
         - If still failing after retries:
             - Does this file have dependents? → halt split, write to forge-coordination.md, ask user
             - Is this file isolated?          → flag in forge-coordination.md, mark as skipped, continue
             - Is this file critical?          → halt split, write to forge-coordination.md, ask user

    4. CHECK DEPENDENCY GRAPH PROGRESS
       Mark completed files as done in agent status map.
       **DAG re-validation:** Before marking a split complete, re-validate the dependency graph against the actual code changes. If code agents introduced new imports or cross-file references that create a dependency cycle, STOP the split and report the cycle to the user via forge-coordination.md. Do not proceed to verifiers until the cycle is resolved.
       If unblocked files remain → go back to step 2 (next dependency round).
       If all files in split are done or skipped → proceed to step 5.

    5. SPAWN VERIFIERS (Task tool, with write isolation)

       Each verifier writes to its OWN isolated file — never to the shared coordination file.

       **Complexity check**: Read `complexity` from forge-state.md.

       **Safety check (large tasks only)**: If architecture doc is missing at this point AND complexity is large, STOP and return to Phase 1.

       - **Plan Verifier**→ runs EVERY iteration (always, regardless of complexity)
         ```
         task(agent_type="foundry/plan-verifier", prompt="
           Plan doc: [full plan document]
           Per-file diffs: [diffs]
           File list: [files in this split]
           Split description: [split N details]
           Split number: [N of total splits]
           Iteration number: [current RALPH iteration]
           Output file: forge-verifier-plan.md (located at `<actual $FOUNDRY_DIR path>`)
         ")
         ```

       - **Architecture Verifier** → runs only at SPLIT COMPLETION (all files done), **LARGE tasks only**
         If `complexity: small` in forge-state.md, skip entirely. Log: `⚡ Architecture verifier skipped — small task (per Crucible complexity assessment)`
         ```
         task(agent_type="foundry/architecture-verifier", prompt="
           Architecture doc: [full architecture document]
           Per-file diffs: [diffs]
           File list: [files in this split]
           Split description: [split N details]
           Split number: [N of total splits]
           Iteration number: [current RALPH iteration]
           Output file: forge-verifier-arch.md (located at `<actual $FOUNDRY_DIR path>`)
         ")
         ```

       - **Design Verifier** → runs only at SPLIT COMPLETION (all files done), **LARGE tasks only**
         If `complexity: small` in forge-state.md, skip entirely. Log: `⚡ Design verifier skipped — small task (per Crucible complexity assessment)`
         If no design doc was provided AND complexity is large, this is a **hard stop** — a large task MUST have a design doc for verification. Log: `❌ FATAL: Design doc missing for large task — aborting execution. Re-run Crucible to generate the design doc, then invoke Forge again.` Set `forge-state.md → status: failed-design-doc-required`. Write failure reason to Failure Log section of `forge-task-log.md`. Exit the RALPH loop immediately — do NOT continue with remaining splits or deep review. Present error summary to user:
         ```
         ❌ Forge stopped — design doc required for large task.
           Status   : failed-design-doc-required (saved to forge-state.md)
           Action   : Re-run Crucible to generate a design doc, then invoke Forge again.
           State    : forge-state.md preserved for resume after design doc is provided.
         ```
         If no design doc was provided AND complexity is small, this verifier was already skipped above.
         ```
         task(agent_type="foundry/design-verifier", prompt="
           Design doc: [full design document]
           Per-file diffs: [diffs]
           File list: [files in this split]
           Split description: [split N details]
           Split number: [N of total splits]
           Iteration number: [current RALPH iteration]
           Output file: forge-verifier-design.md (located at `<actual $FOUNDRY_DIR path>`)
         ")
         ```

       Generate per-file diffs (not full-repo diff):

       Write diffs to `$FOUNDRY_DIR` (never to the repo working directory):
       - All forge working files MUST live in `$FOUNDRY_DIR` — see Rule at line ~1091
       - NEVER hardcode `/tmp/` — this fails on Windows

       ```powershell
       # PowerShell — write verifier diffs to $FOUNDRY_DIR (not repo root)
       # Detects PowerShell version for BOM-free UTF-8 encoding
       function Write-Utf8NoBom($path, $content) {
           if ($PSVersionTable.PSVersion.Major -ge 7) {
               $content | Out-File -Encoding utf8NoBOM $path
           } else {
               [IO.File]::WriteAllText($path, $content, [Text.UTF8Encoding]::new($false))
           }
       }
       # Stage new (CREATE) files so git diff HEAD can see them
       git add --intent-to-add <file1> <file2>  # for all files assigned to this split
       Write-Utf8NoBom "$FOUNDRY_DIR/forge-diff-file1.patch" (git diff HEAD -- <file1> | Out-String)
       Write-Utf8NoBom "$FOUNDRY_DIR/forge-diff-file2.patch" (git diff HEAD -- <file2> | Out-String)
       ```
       ```bash
       # Bash — write verifier diffs to $FOUNDRY_DIR (not repo root)
       # Stage new (CREATE) files so git diff HEAD can see them
       git add --intent-to-add <file1> <file2>  # for all files assigned to this split
       git diff HEAD -- <file1> > "$FOUNDRY_DIR/forge-diff-file1.patch"
       git diff HEAD -- <file2> > "$FOUNDRY_DIR/forge-diff-file2.patch"
       ```

       Wait for ALL active verifiers to complete.

       **Consolidation**: After all verifiers finish, read each `forge-verifier-*.md` file and consolidate into `forge-coordination.md`:

       ### Coordination File Update

       **Append, do not overwrite** `forge-coordination.md`. Each iteration adds a new section:

       ```markdown
       ---
       ## Split [N] — Iteration [I]
       [timestamp]

       ### Consolidated Verifier Feedback

       #### Plan Verifier
       [copied from forge-verifier-plan.md]

       #### Architecture Verifier
       [copied from forge-verifier-arch.md, or "SKIPPED — runs at split completion", or "SKIPPED — small task (per Crucible complexity assessment)"]

       #### Design Verifier
       [copied from forge-verifier-design.md, or "SKIPPED — runs at split completion", or "SKIPPED — small task (per Crucible complexity assessment)"]

       ### Files Status
       [current status of each file]
       ---
       ```

       Code agents read ONLY the latest section (last `---` delimited block). Full history is preserved for debugging and scribe reference.

       If the file exceeds 50 sections (count `---` delimiters at column 0, outside fenced code blocks — per Crucible Rule 10), archive older sections to `forge-coordination-archive.md` and keep only the last 10 in the active file.

    6. SCRIBE (conditional)

       **Timing:** Scribe is dispatched AFTER an iteration fully completes (all code agents finish, all verifiers return, coordination file is updated). Never dispatch Scribe mid-iteration — it must see the final state.

       Invoke Scribe:
         - If ANY verifier returned ISSUES FOUND
         - If this is the final approved iteration of a split
         - If this is a deep review iteration

       Skip Scribe on clean mid-split iterations where all active verifiers approve
       and there's nothing notable to record. When skipping, log: `ℹ️ Scribe skipped — all verifiers approved, no notable changes this iteration.`

       Scribe reads:
         - `forge-coordination.md` (consolidated verifier feedback)
         - Per-file diffs from this iteration
         - Current agent status map

       Dispatch:
       ```
       task(agent_type="foundry/scribe", prompt="
         Coordination file: [contents of forge-coordination.md]
         Per-file diffs: [diffs from this iteration]
         Agent status: [current agent status map]
         Complexity: [from forge-state.md — small or large]
         Record: iteration number, files changed, verifier verdicts, decision made
         Output: append one log entry to forge-task-log.md
         Write all log files to $FOUNDRY_DIR/ (value: <actual $FOUNDRY_DIR path>)
       ")
       ```

       Scribe entry must include:
       - Iteration number
       - Files changed
       - Verifier verdicts
       - Decision made (continue/commit/retry)

       Scribe appends one log entry to forge-task-log.md.

    7. EVALUATE ITERATION OUTCOME
       All active verifiers APPROVED?
         YES → proceed to PER-SPLIT DEEP REVIEW (step 8)
         NO  → iteration count < hard cap?
                 YES → clear APPROVED sections in forge-coordination.md,
                        keep ISSUES sections → next iteration
                 NO  → HARD CAP REACHED (see below)

       **File status reset on failure**: When verifiers report ISSUES FOUND for specific files:
       1. For each file flagged with issues in `forge-coordination.md`, reset its status in `forge-state.md` from `DONE` to `PENDING`
       2. This ensures Step 2 (SPAWN CODE AGENTS) re-spawns agents for these files in the next iteration
       3. **Max retry per file**: Track retry count per file in `forge-state.md`. If a file fails 3 consecutive iterations:
          - Mark it as `STUCK` in forge-state.md
          - Add it to the issues list in `forge-coordination.md` with all 3 failure reasons
          - Continue with remaining files — do not block the entire split
          - Present STUCK files to the user at split completion for manual intervention

    8. PER-SPLIT DEEP REVIEW

       Once all verifiers approve for this split, run deep review BEFORE committing.
       This catches issues early — before they propagate to the next split.

       Generate the split's diff scoped to THIS SPLIT'S FILES ONLY using BOM-free UTF-8 encoding (see § Verifier Diff Generation above for the `Write-Utf8NoBom` helper):
       ```powershell
       # PowerShell — write deep review diff to $FOUNDRY_DIR (scoped to split's files)
       Write-Utf8NoBom "$FOUNDRY_DIR/forge-deep-review-diff.patch" (git diff HEAD -- <file1> <file2> <fileN> | Out-String)
       ```
       ```bash
       # Bash — write deep review diff to $FOUNDRY_DIR (scoped to split's files)
       git diff HEAD -- <file1> <file2> <fileN> > "$FOUNDRY_DIR/forge-deep-review-diff.patch"
       ```

       **Large diff handling**: If the diff exceeds ~80,000 characters:
       1. Split into per-file chunks
       2. Group into batches that fit within context (~80k chars each)
       3. Dispatch one set of 3 deep-review agents PER BATCH
       4. Synthesize results across batches

       Dispatch three parallel agents using the `deep-review` plugin (NOT the built-in `code-review` agent type):
       ```
       task(agent_type="deep-review:architect", prompt="Review this diff for direction and design soundness:\n\n[diff contents]")
       task(agent_type="deep-review:advocate", prompt="Defend this implementation — find strengths and justify decisions:\n\n[diff contents]")
       task(agent_type="deep-review:skeptic", prompt="Attack this code — find flaws, risks, and weaknesses:\n\n[diff contents]")
       ```

       > ⚠️ **MANDATORY**: Always use `deep-review:architect`, `deep-review:advocate`, and `deep-review:skeptic` agent types from the `deep-review` plugin. Do NOT substitute with the built-in `code-review` agent type. If `code-review` was run separately (e.g., during RALPH), it does NOT replace the deep-review step — deep-review MUST still run. The deep-review loop continues until all three perspectives report no CRITICAL issues.

       Wait for all three to complete. Write findings to `forge-coordination.md` under `## Deep Review — Split [N] Round [R]`.

       **Evaluate:**
       - All three satisfied with no CRITICAL issues? → Invoke Scribe with "Deep Review Complete" entry for this split, then proceed to BUILD GATE (step 9)
       - CRITICAL issues found? → address feedback:
         1. Triage findings, group by file
         2. Identify conflicts between perspectives — flag and ask user
         3. Re-enter RALPH loop (step 2) for affected files only (max 3 fix iterations per deep-review round — these do NOT count against the 10-iteration RALPH cap since verifiers already approved)
         4. After fixes, re-run deep review (regenerate diff, dispatch agents)
         5. Increment `deep-review-round` in `forge-state.md`
       - Hard cap: 5 deep review rounds per split (each with up to 3 fix iterations). If reached, pause and present remaining issues to user.

       Update `forge-state.md`: `deep-review-round` after each round.

    9. PER-SPLIT BUILD GATE

       After deep review passes, run a full project build BEFORE committing.

       Auto-detect build system:
       ```
       Detection order:
       1. If `dev build` is available (CoreXT repo)     → dev build
       2. If package.json exists with build script       → npm run build
       3. If *.csproj or *.sln exists                    → dotnet build
       4. If Makefile exists                             → make
       5. If Cargo.toml exists                           → cargo build
       6. If go.mod exists                               → go build ./...
       7. No build system detected                       → skip with warning
       ```

       If build **FAILS**:
       - Write build errors to `forge-coordination.md`
       - Re-enter RALPH loop (step 2) to fix build errors
       - After fixes, re-run build (no need to re-run deep review unless files changed significantly)
       - **Progress check:** Track the primary error signature (first compiler error message) AND total error count across consecutive build-fix attempts. If the SAME primary error persists for 3 consecutive attempts OR the total error count has not decreased for 3 consecutive attempts, escalate immediately to the user — do not wait for the hard cap. Log: `⚠️ Build fix loop stalled — same error after 3 attempts. Escalating.`
       - Repeat until build passes — hard cap of 5 build-fix attempts (tracked in forge-state.md → build-fix-attempts) to prevent infinite loops. If build still fails after 5 attempts, present to user with options (similar to RALPH hard cap menu).

       If build **PASSES** → proceed to COMMIT AND PUSH (step 10)

    10. COMMIT AND PUSH

       Stage ONLY the specific files assigned to this split's agents:
       ```bash
       git add <file1> <file2> <file3>
       ```

       Do NOT use `git add .` or `git add -A`. Only stage files that code agents were assigned to modify.

       Before pushing, rebase against the parent branch:
       ```bash
       git fetch origin
       ```

       For **single-branch mode** — rebase against the base branch:
       ```bash
       git rebase origin/$BASE_BRANCH
       ```

       For **multi-split, split 1** — rebase against the user-specified base branch:
       ```bash
       git rebase origin/$BASE_BRANCH
       ```

       For **multi-split, split N > 1** — rebase target depends on split relationship:
       - If `split-relationship: chained` → rebase against the previous split's branch:
       ```bash
       git rebase origin/<task-branch>/split-<N-1>
       ```
       - If `split-relationship: independent` → rebase against the base branch (same as split 1):
       ```bash
       git rebase origin/$BASE_BRANCH
       ```

       If rebase **fails** (conflicts):
       - `git rebase --abort`
       - PAUSE execution
       - Report to user: "Rebase conflict detected. Please resolve manually and run 'continue'."
       - Do NOT force-push or auto-resolve

       Write a commit message that:
       - Describes what was implemented (not "iteration N" or "split N")
       - References the specific behavior or feature delivered
       - Is concise and meaningful
       - Follows the host environment's commit trailer policy. If the environment requires `Co-authored-by` or other trailers, include them. If no policy exists, omit agent signatures.

       Commit and push to the current branch:
       ```bash
       # Single-branch mode:
       git commit -m "<meaningful message>"
       git push origin <task-branch>

       # Multi-split mode:
       git commit -m "<meaningful message>"
       git push origin <task-branch>/split-N
       ```

    ### Rollback & Retry Strategy

    **Git checkpoint**: Before starting each split's execution:
    1. Create a checkpoint tag using a slugified branch name (replace `/` with `-`) to prevent git ref path conflicts:
       - Single: `git tag forge-checkpoint--<task-branch-slugified>`
       - Multi: `git tag forge-checkpoint--<task-branch-slugified>--split-N`
       Example: branch `forge/my-task/split-1` → tag `forge-checkpoint--forge-my-task--split-1`
       Example: single branch `user/johndoe/fix-bug` → tag `forge-checkpoint--user-johndoe-fix-bug`
    2. If the split fails after 10 iterations (hard cap), offer:
       - Revert to checkpoint: `git reset --hard <checkpoint-tag>` (reset is preferred over revert because checkpoint tags always mark committed states — there are no uncommitted changes to lose, and reset avoids the failure mode where `git revert --no-commit` chokes on a dirty working tree)
       - Keep partial work and continue to next split
       - Stop and let user intervene

    **Agent dispatch retry**: If a code agent or verifier fails to respond (timeout, API error, empty response):
    1. Retry once with the same model after 10 seconds
    2. If retry fails, try with fallback model (`claude-sonnet-4.6` as universal fallback)
    3. If fallback fails, mark the file as `STUCK` and continue with other files
    4. Log all failures to `forge-coordination.md`

SPLIT END→ clay is fired, move to next split (next piece on the wheel)
```

> **Watching the loop is where you learn.** When you see a failure domain — a verifier
> that keeps rejecting, a build that won't pass, a dependency that blocks everything —
> put on your engineering hat and resolve the problem so it never happens again.
> The RALPH loop is not just an execution pattern; it's a learning pattern.

### Hard Cap Behavior (Knowing When to Stop the Wheel)

When the 10-iteration hard cap is reached without all verifiers approving — the clay has been on the wheel too long:

1. **Do NOT silently stop or auto-resolve.**
2. Summarize progress:
   ```
   ## Hard Cap Reached — Split [N]

   Iterations completed: 10 / 10
   Files completed: [N / total]
   Last verifier status:
     - Plan        : [APPROVED / ISSUES]
     - Architecture: [APPROVED / ISSUES / not yet run]
     - Design      : [APPROVED / ISSUES / not yet run]

   Open issues:
   [numbered list from forge-coordination.md]

   Options:
   1. Continue for 5 more iterations (extends cap to 15)
   2. Revert to last good commit and pause
   3. Abandon this split and move to next
   4. Stop entirely — I'll review manually
   ```
3. Wait for user decision. Do not proceed without explicit input.
4. **If user chooses option 1 (extend)**: Update `hard-cap-iterations` in `forge-state.md` to the new value (15) before resuming. This ensures resume-safety — if the session is interrupted during the extended run, the state file reflects the correct cap. Invoke Scribe to log the extension decision before continuing: `task(agent_type="foundry/scribe", prompt="Log hard cap extension: user chose to extend from 10 to 15 iterations for split N. Reason: [user's stated reason or 'no reason given']")`. (If the extended cap is also reached, the same menu is presented again — the user can extend further in increments of 5, or choose any other option.)

---

### Completion

After all splits are done (each split has passed verifiers, deep review, and build gate):

1. Invoke Scribe to write a final summary to `forge-task-log.md` (use the Scribe agent's `## Task Complete` template — do NOT use an inline template here; Scribe is the single source of truth for entry formatting):
   ```
   task(agent_type="foundry/scribe", prompt="Write the Task Complete entry. Strategy: [single|multi], Complexity: [small|large], Splits: [N/N], Commits: [N], Summary: [what was built, key decisions]")
   ```

2. Proceed directly to Phase 7 (Summary and Cleanup).

---

## Phase 7 — Summary and Cleanup

All splits are complete. Each split has been deep-reviewed, build-verified, committed, and pushed.

### Step 1 — Write Execution Summary

Write execution summary to `forge-summary.md`:

**Single-branch mode:**
   ```markdown
   # Forge Execution Summary

   ## Task
   [task title from plan]

   ## Complexity
   [small | large]

   ## Split Strategy
   single — all changes on one branch

   ## Branch
   | Branch | Commits | Status |
   |--------|---------|--------|
   | <task-branch> | [N] | ✅ Complete |

   ## Base Branch
   [base branch name]

   ## Deep Review
   Rounds: [N]
   Result: All perspectives satisfied

   ## Build Gate
   Result: [PASSED / FAILED + details]

   ## Files Changed
   [total files]

   ## Task Log
   See `forge-task-log.md` in the Forge working directory for iteration-by-iteration details.

   ## Working Directory
   [full $FOUNDRY_DIR path]

   ## Next Steps
   1. Review the code on the branch
   2. Create a PR from `<task-branch>` → `<base-branch>`
   3. Delete the branch after merge
   ```

**Multi-split mode:**
   ```markdown
   # Forge Execution Summary

   ## Task
   [task title from plan]

   ## Complexity
   [small | large]

   ## Split Strategy
   multi — [N] splits across separate branches

   ## Branches
   | Split | Branch | Commits | Status |
   |-------|--------|---------|--------|
   | Split 1 — [name] | <task-branch>/split-1 | [N] | ✅ Complete |
   | Split 2 — [name] | <task-branch>/split-2 | [N] | ✅ Complete |

   ## Base Branch
   [base branch name]

   ## Merge Order
   <!-- Conditional: show appropriate guidance based on split relationship -->
   [If chained]: Merge in order: split-1 → split-2 → ... → split-N. Each split branch chains from the previous.
   [If independent]: Merge in any order — splits are independent, each branched from base.

   ## Deep Review
   Rounds: [N]
   Result: All perspectives satisfied

   ## Build Gate
   Result: [PASSED / FAILED + details]

   ## Files Changed
   [total files across all splits]

   ## Task Log
   See `forge-task-log.md` in the Forge working directory for iteration-by-iteration details.

   ## Working Directory
   [full $FOUNDRY_DIR path]

   ## Next Steps
   1. Review the code on each split branch
   2. Create PRs from each split branch (merge in order)
   3. Delete split branches after merge
   ```

3. Report to user (brief — details are in the summary file):

   **Single-branch:**
   ```
   ✅ Forge complete.

   Summary  : [$FOUNDRY_DIR/forge-summary.md]
   Branch   : <task-branch>
   Log      : [$FOUNDRY_DIR/forge-task-log.md]

   Create a PR from <task-branch> → <base-branch> when ready.
   ```

   **Multi-split:**
   ```
   ✅ Forge complete.

   Summary  : [$FOUNDRY_DIR/forge-summary.md]
   Branches : [list split branches]
   Log      : [$FOUNDRY_DIR/forge-task-log.md]

   Create PRs from each split branch when ready — merge in order.
   ```

4. Auto-cleanup (no prompts):

   **Coordination file cleanup:** Before deleting working files, archive `forge-coordination.md` entries. If the file exists, keep only the last 2 iteration sections (by `---` delimiters) in the active file. Move older sections to `forge-coordination-archive.md` in `$FOUNDRY_DIR`. This preserves recent context for debugging while preventing unbounded growth.

   Delete all forge working files in `$FOUNDRY_DIR` except outputs (`forge-summary.md`, `forge-task-log.md`, and `forge-coordination-archive.md`):

   ```powershell
   # PowerShell — clean up working files, keep outputs
   Get-ChildItem -Path $FOUNDRY_DIR -Filter "forge-*" |
     Where-Object { $_.Name -notmatch '^forge-(summary|task-log|coordination-archive)\.md$' } |
     Remove-Item -Force
   ```
   ```bash
   # Bash
   find "$FOUNDRY_DIR" -maxdepth 1 -name 'forge-*' ! -name 'forge-summary.md' ! -name 'forge-task-log.md' ! -name 'forge-coordination-archive.md' -delete
   ```

   Log what was cleaned: `Cleaned up [N] working files in $FOUNDRY_DIR. Kept forge-summary.md, forge-task-log.md, and forge-coordination-archive.md.`

   > **Note**: The `forge-*` glob intentionally excludes `plan.md`, `design-doc.md`, and Crucible outputs — these are preserved as task artifacts since they don't match the glob pattern.

5. Auto-cleanup checkpoint tags for THIS task only (no prompts):

   ```powershell
   # PowerShell — scope to current task slug to avoid deleting other sessions' tags
   $taskBranchSlug = "<slugified-task-branch>"  # NOTE: checkpoint tags use slugified BRANCH name, not task-name slug
   # Use precise glob: add -- delimiter to prevent matching other tasks with similar prefixes
   git tag -l "forge-checkpoint--${taskBranchSlug}" -l "forge-checkpoint--${taskBranchSlug}--split-*" | ForEach-Object {
       $pushed = git push origin --delete $_ 2>$null
       if ($LASTEXITCODE -ne 0) {
           Write-Host "⚠️ Failed to delete remote tag $_  — will retry once"
           Start-Sleep -Seconds 2
           git push origin --delete $_ 2>$null  # single retry
       }
       git tag -d $_
   }
   ```
   ```bash
   # Bash — scope to current task slug
   TASK_SLUG="<slugified-task-branch>"
   # Use precise glob: match exact single-branch tag OR --split-* suffix to prevent cross-task deletion
   { git tag -l "forge-checkpoint--${TASK_SLUG}" ; git tag -l "forge-checkpoint--${TASK_SLUG}--split-*" ; } | sort -u | while read -r tag; do
       if ! git push origin --delete "$tag" 2>/dev/null; then
           echo "⚠️ Failed to delete remote tag $tag — will retry once"
           sleep 2
           git push origin --delete "$tag" 2>/dev/null  # single retry
       fi
       git tag -d "$tag"
   done
   ```

   **Atomicity note:** Always delete the remote tag BEFORE the local tag for resume safety. If the remote push fails, the local tag is still present so the tag is discoverable on retry. Never delete local tags in bulk without attempting the remote push first — a partial cleanup where remote tags are gone but local tags remain is recoverable; the reverse is not.

   Log: `Cleaned up [N] checkpoint tags.`

Do not merge. Hand off to the user.

---

## Rules (Never Violate)

- Never write or modify code before Phase 5 user confirmation
- Never proceed past a MISSING critical item without user acknowledgment
- Never spawn file agents for files with unresolved dependencies
- Code agents must only modify their assigned file — never touch other files
- Always read forge-state.md at the start of every iteration — this is the source of truth
- Always write forge-state.md after every step — interrupted forge must be resumable
- Always read forge-coordination.md fresh from disk at each iteration start
- Always set a hard cap per split (default 10 iterations) — never loop without a ceiling
- If hard cap is reached, always present options to user — never auto-resolve
- Never create PRs — Forge pushes to the remote branch only. PR creation is the user's responsibility.
- Always push immediately after every commit
- Always fetch and rebase before push — never push without checking for drift
- On rebase conflict, always pause and ask user — never force-push or auto-resolve
- Always stage only specific assigned files — never use `git add .` or `git add -A`
- Each verifier writes to its own file — never have multiple verifiers write to the same file
- Pass document summaries to code agents — never send full documents to every agent
- Follow the host environment's commit trailer policy — never impose or ban specific trailers
- All forge working files live in `$FOUNDRY_DIR` (~/.copilot/foundry/<task-slug>/) alongside Crucible outputs — never in the repo
- Always flag conflicting feedback between deep review perspectives and ask user before resolving
- Always cap deep review rounds at 5 — if not satisfied by then, pause and ask user
- Architecture doc is required for LARGE tasks — if not provided, block execution and ask the user. For SMALL tasks (as classified by Crucible), skip architecture and design doc requirements entirely.
- Build gate and deep review run per-split (after verifiers approve, before commit) — not after all splits.
- Deep review runs locally against the split's uncommitted diff — no PR required.
- Always read base branch from the plan's `## Execution Config` section — never auto-detect or hardcode main/master. If execution config is missing, fall back to prompting the user once.
- Read split-relationship (chained/independent) from the plan's `## Execution Config` section; only prompt if the section is missing — independent splits should be separate Forge executions.
- Always dispatch agents explicitly using task(agent_type='foundry/<agent-name>') — never use vague instructions.
- Each split gets its own branch (<task-branch>/split-N), chained from the previous split OR from the base branch (if independent) — UNLESS `split-strategy` is `single`, in which case use `<task-branch>` directly without `/split-N` suffix.
- Read branch naming preference/prefix from the plan's `## Execution Config` section; only prompt if the section is missing — never duplicate the prompt in Phase 6.
- Always check for an existing branch (local → remote) BEFORE creating a new one from a parent — this prevents resume divergence.
- When creating split-N branches, always include `origin/<task-branch>/split-<N-1>` as a final fallback parent for fresh-environment resume.
- Always provide both PowerShell and Bash variants for shell commands. Never use bash-only syntax (e.g., `2>/dev/null`) in PowerShell blocks — use `2>$null` or try/catch instead.
- Generate diff using utf8NoBOM encoding (see § Verifier Diff Generation above for encoding details).
- Always slugify branch names in checkpoint tags (replace `/` with `-`) to prevent git ref path conflicts with hierarchical branch names.
- When extending the hard cap, always persist the new value to `hard-cap-iterations` in `forge-state.md` before continuing.
- When the deep review diff exceeds ~80k characters, chunk it into per-file batches and run parallel deep review agents per batch, then synthesize.
- Always clean up checkpoint tags and working files at task completion.
- Deep review MUST use the `deep-review` plugin agents (`deep-review:architect`, `deep-review:advocate`, `deep-review:skeptic`) — NEVER substitute with the built-in `code-review` agent type. If `code-review` was run independently, deep-review still runs. The deep-review loop is mandatory and continues until all three perspectives report no CRITICAL issues.
