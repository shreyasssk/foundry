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

Before asking for documents, check if a `forge-state.md` already exists in the working directory or recent project folders:

1. Search for `forge-state.md` in the current directory and immediate subdirectories
2. If found, read it and validate:
   - Does the branch still exist? (`git branch --list <branch>`)
   - Are the split/iteration counters consistent?
   - Is the working tree clean? (`git status --porcelain`)
   - Does `base-branch` exist? If missing (legacy state from pre-v1.3.5), ask the user to provide it before resuming.
   - Does `complexity` exist? If missing (legacy state from pre-v1.4.0), default to `large` and log: `⚠️ Pre-v1.4.0 state detected — assuming large task (full ceremony). Override with plan's ## Complexity section if present.`
3. If valid, present to the user:
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
   2. Start fresh (will archive old state)
   ```
4. If the user chooses to resume, skip to the phase indicated in `forge-state.md`.
5. If state is invalid (branch deleted, inconsistent counters), warn and recommend starting fresh.

### New Task Intake

If no state found or user chose fresh start, ask for document locations in a single message:

```
To get started with Forge, please provide paths to the following.
You can give a file path or a directory (I'll find the right files inside).

1. Plan               — REQUIRED (file or folder path)
2. Architecture doc   — required for large tasks (file or folder path)
3. Design doc         — required for large tasks (file or folder path)
4. Product spec       — optional (skip if not applicable)
5. Task description   — brief summary or work item ID
```

**Complexity-aware intake:** After reading the plan, check for a `## Complexity` section:
- If `Classification: small` is found → this is a Crucible-assessed small task. **Do NOT prompt for design doc or architecture doc.** Display:
  ```
  ⚡ Detected complexity: SMALL (from Crucible plan)
     — Skipping design doc requirement
     — Skipping architecture doc requirement
     Proceeding with plan-only execution.
  ```
- If `Classification: large` is found (or no complexity section exists) → treat as a large task requiring full ceremony (architecture doc required, design doc expected)

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
- Is the file dependency graph acyclic? (Validate DAG — flag cycles as blocking errors)

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
  - If missing → `[MISSING]` warn, offer to create splits yourself or ask user to add them
- [ ] Branch name is specified
  - If missing → `[MISSING]` warn, offer to define one yourself or ask user
- [ ] File-level breakdown exists per split
  - If missing → `[MISSING]` warn — execution cannot safely spawn per-file agents without this
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

Write these summaries to the project folder. Code agents receive summaries; verifiers receive full documents.

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

## Phase 5 — Final Confirmation

Present a concise execution preview. Before showing the preview, collect three key decisions from the user:

**1. Base branch:** Ask the user which branch to create splits from:

```
What branch should I create the first split from?

This is the base branch that split-1 will branch off of.
Common choices: main, master, develop, build/main/latest

Your base branch?
```

Store this as `$BASE_BRANCH` — used instead of auto-detection throughout.

**2. Branch naming preference:** If the plan specifies a branch name, use it. If not (or the user wants to customize), ask:

```
Branch naming — what prefix should I use for split branches?

Common patterns:
  1. user/<alias>/<task-name>/split-N    (e.g., user/johndoe/my-feature/split-1)
  2. feature/<task-name>/split-N          (e.g., feature/add-auth/split-1)
  3. forge/<task-name>/split-N            (e.g., forge/add-auth/split-1)
  4. Custom prefix

Your preference? (or press Enter for option 3)
```

**3. Split relationship:** If the plan has multiple splits, ask:

```
Are the splits in this task chained (each builds on the previous)?

  1. Yes — chained (split-2 branches from split-1, split-3 from split-2, etc.)
     Use this when all splits are part of the same feature/change.
  2. No — they are independent/unrelated changes

Your choice?
```

If the user chooses **independent**: Advise them to run Forge separately for each split — independent changes should not share a branch chain. Offer to proceed with just the first split, or let the user pick which split to execute.

If the user chooses **chained**: Proceed with the normal per-split branching flow.

Once all decisions are confirmed, show the full execution preview:

```
## Ready to Forge

Base     : [base branch]
Branch   : [confirmed branch prefix]/split-1..N
Splits   : [N splits — list them with their file counts]
Chaining : [chained / independent]
Strategy : One code agent per file, parallel within dependency constraints
           Verifiers (plan every iteration; architecture + design at split completion [large tasks] or SKIPPED [small tasks])
           Build gate after all splits complete, before deep review
           Scribe maintains task log at [project folder path]

Shall I proceed?
```

Wait for explicit user confirmation. Do not touch any file until confirmed.

---

## Phase 6 — Execution

### Setup

Before starting the loop:

#### 1. Environment Preflight Checks

```bash
# Verify clean working tree
git status --porcelain
# If output is non-empty → STOP, ask user to stash or commit

# Verify remote is reachable
git ls-remote --exit-code origin
# If fails → STOP, ask user to check network/auth

```

If any preflight check fails, do NOT proceed. Report the failure and wait for the user.

#### 2. Forge File Hygiene

Ensure forge artifacts don't pollute version control. Check for BOTH patterns (`.md` and `.patch`) — an existing `.gitignore` may have an older forge entry that only covers `.md`:

**Windows (PowerShell):**
```powershell
$gitignore = if (Test-Path .gitignore) { Get-Content .gitignore -Raw } else { '' }
$needsMd = $gitignore -notmatch 'forge-\*\.md'
$needsPatch = $gitignore -notmatch 'forge-\*\.patch'
if ($needsMd -or $needsPatch) {
    $additions = "`n# Forge orchestration artifacts"
    if ($needsMd) { $additions += "`nforge-*.md" }
    if ($needsPatch) { $additions += "`nforge-*.patch" }
    Add-Content -Path .gitignore -Value $additions
}
```

**Unix (bash):**
```bash
touch .gitignore
grep -q 'forge-\*\.md' .gitignore || echo -e '\n# Forge orchestration artifacts\nforge-*.md' >> .gitignore
grep -q 'forge-\*\.patch' .gitignore || echo 'forge-*.patch' >> .gitignore
```

If `.gitignore` doesn't exist, create it with the forge pattern. If modifying `.gitignore`, stage and commit it immediately with message: `chore: add forge artifacts to .gitignore`.

#### 3. Create Coordination Files

Create the coordination directory in the project folder (same folder as plan/design doc):

```
<project-folder>/
├── forge-state.md               ← orchestrator loop state
├── forge-coordination.md        ← consolidated verifier feedback
├── forge-verifier-plan.md       ← plan verifier output (isolated)
├── forge-verifier-arch.md       ← architecture verifier output (large tasks only)
├── forge-verifier-design.md     ← design verifier output (large tasks only)
├── forge-task-log.md            ← scribe log
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
Started: [ISO timestamp]
Splits: [N]
Branching: per-split (<task-branch>/split-1, split-2, ...)
---
```

#### 5. Checkout or Create Branch (per-split branching)

Each task split gets its own branch, chained from the previous split's branch. This keeps all splits under the same root task while allowing independent review per split.

Use the branch prefix and base branch confirmed by the user in Phase 5. The split suffix `/split-N` is always appended automatically. Store `$BASE_BRANCH` from the user's Phase 5 answer.

**Branch naming convention:** `<task-branch>/split-N` (e.g., `user/johndoe/my-feature/split-1`, `feature/my-task/split-2`)

**Important:** The task branch name (`<task-branch>`) must NOT be an existing branch name. Use a path-style name like `user/<alias>/<task-name>` or `feature/<task-name>` — this ensures `<task-branch>/split-N` won't collide with existing refs. If `<task-branch>` exists as a branch, prefix it: `forge/<task-branch>`.

```bash
git fetch origin
```

**Split 1** — branch from the user-specified base branch (prefer existing if resuming):
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

**Split N (N > 1)** — branch from the previous split (with full fallback chain):
```powershell
# PowerShell — full fallback chain for resume safety
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
# Bash — full fallback chain for resume safety
git checkout <task-branch>/split-N 2>/dev/null \
  || git checkout -b <task-branch>/split-N origin/<task-branch>/split-N 2>/dev/null \
  || git checkout -b <task-branch>/split-N <task-branch>/split-<N-1> 2>/dev/null \
  || git checkout -b <task-branch>/split-N origin/<task-branch>/split-<N-1>
```

Store the current split branch name in `forge-state.md` under `current-branch`.

#### 6. Initialize State File

Create `forge-state.md`:

```markdown
---
split: 1
iteration: 1
hard-cap-iterations: 10
deep-review-round: 0
hard-cap-deep-review: 5
status: running
phase: execution
complexity: <small|large>  ← read from plan.md's ## Complexity → Classification field
task-branch: <task-branch>
base-branch: <base-branch>
current-branch: <task-branch>/split-1
chained: <true|false>
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

## Deep Review
- Round       : 0
- Last result : —

## Failure Log
(none)
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

Repeat for each split in order:

**Important:** If the user chose "independent" splits in Phase 5, only ONE split should be executing in this Forge session. If multiple splits somehow reached Phase 6 with `chained: false` in forge-state.md, STOP and remind the user that independent splits require separate Forge executions.

```
SPLIT START (put the clay on the wheel)
  Create split branch:
    If split 1 → already on <task-branch>/split-1 from setup
    If split N > 1 (full fallback chain — check existing first, then create):
      Bash: git checkout <split-N> || git checkout -b <split-N> origin/<split-N> || git checkout -b <split-N> <split-N-1> || git checkout -b <split-N> origin/<split-N-1>
      PowerShell: same order — local → origin/self → local-parent → origin/parent
  Update forge-state.md: current-branch = <task-branch>/split-N
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
         Instructions: Implement only what is needed for this file in this split. Do not commit. Do not modify any other file.
       ")
       ```

       Wait for ALL code agents to complete before continuing.

    3. HANDLE CODE AGENT FAILURES
       For each failed agent:
         - Retry up to 2 times
         - If still failing after retries:
             - Does this file have dependents? → halt split, write to forge-coordination.md, ask user
             - Is this file isolated?          → flag in forge-coordination.md, mark as skipped, continue
             - Is this file critical?          → halt split, write to forge-coordination.md, ask user

    4. CHECK DEPENDENCY GRAPH PROGRESS
       Mark completed files as done in agent status map.
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
           Output file: forge-verifier-plan.md
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
           Output file: forge-verifier-arch.md
         ")
         ```

       - **Design Verifier** → runs only at SPLIT COMPLETION (all files done), **LARGE tasks only**
         If `complexity: small` in forge-state.md, skip entirely. Log: `⚡ Design verifier skipped — small task (per Crucible complexity assessment)`
         If no design doc was provided AND complexity is large, this should not happen — readiness should have blocked. Log a warning: `⚠️ Design doc missing for large task — this should have been caught in Phase 4 readiness. Skipping design verifier.`
         If no design doc was provided AND complexity is small, this verifier was already skipped above.
         ```
         task(agent_type="foundry/design-verifier", prompt="
           Design doc: [full design document]
           Per-file diffs: [diffs]
           File list: [files in this split]
           Split description: [split N details]
           Output file: forge-verifier-design.md
         ")
         ```

       Generate per-file diffs (not full-repo diff):

       Write diffs to the project folder using OS-agnostic paths:
       - Use the project's working directory: `forge-diff-<filename>.patch`
       - Or use the system temp directory: `$env:TEMP` (Windows) / `$TMPDIR` (Unix)
       - NEVER hardcode `/tmp/` — this fails on Windows

       ```powershell
       # PowerShell — use UTF8 encoding (default > produces UTF-16 which breaks git apply)
       # PowerShell — use UTF8NoBOM to avoid BOM issues with git apply
       # PowerShell 7+: Out-File -Encoding utf8NoBOM
       # PowerShell 5.1: [IO.File]::WriteAllText($path, (git diff HEAD -- <file> | Out-String))
       git diff HEAD -- <file1> | Out-File -Encoding utf8NoBOM forge-diff-file1.patch
       git diff HEAD -- <file2> | Out-File -Encoding utf8NoBOM forge-diff-file2.patch
       # Fallback for PowerShell 5.1 (no utf8NoBOM):
       # [IO.File]::WriteAllText("forge-diff-file1.patch", (git diff HEAD -- <file1> | Out-String), [Text.UTF8Encoding]::new($false))
       ```
       ```bash
       # Bash
       git diff HEAD -- <file1> > forge-diff-file1.patch
       git diff HEAD -- <file2> > forge-diff-file2.patch
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

       If the file exceeds 50 sections, archive older sections to `forge-coordination-archive.md` and keep only the last 10 in the active file.

    6. SCRIBE (conditional)

       Invoke Scribe:
         - If ANY verifier returned ISSUES FOUND
         - If this is the final approved iteration of a split
         - If this is a deep review iteration

       Skip Scribe on clean mid-split iterations where all active verifiers approve
       and there's nothing notable to record.

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
         YES → proceed to COMMIT AND PUSH
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

    8. COMMIT AND PUSH

       Stage ONLY the specific files assigned to this split's agents:
       ```bash
       git add <file1> <file2> <file3>
       ```

       Do NOT use `git add .` or `git add -A`. Only stage files that code agents were assigned to modify.

       Before pushing, rebase against the parent branch:
       ```bash
       git fetch origin
       ```

       For **split 1** — rebase against the user-specified base branch:
       ```bash
       git rebase origin/$BASE_BRANCH
       ```

       For **split N > 1** — rebase against the previous split's branch:
       ```bash
       git rebase origin/<task-branch>/split-<N-1>
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

       Commit and push to the current split branch:
       ```bash
       git commit -m "<meaningful message>"
       git push origin <task-branch>/split-N
       ```

    ### Rollback & Retry Strategy

    **Git checkpoint**: Before starting each split's execution:
    1. Create a checkpoint tag using a slugified branch name (replace `/` with `-`) to prevent git ref path conflicts:
       `git tag forge-checkpoint--<task-branch-slugified>--split-N`
       Example: branch `forge/my-task/split-1` → tag `forge-checkpoint--forge-my-task--split-1`
    2. If the split fails after 10 iterations (hard cap), offer:
       - Revert to checkpoint: `git revert --no-commit forge-checkpoint--<task-branch-slugified>--split-N..HEAD && git commit -m "Revert split N (forge rollback)" && git push origin <branch-name>`
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
4. **If user chooses option 1 (extend)**: Update `hard-cap-iterations` in `forge-state.md` to the new value (15) before resuming. This ensures resume-safety — if the session is interrupted during the extended run, the state file reflects the correct cap.

---

### Completion

After all splits are done:

1. Scribe writes a final summary to `forge-task-log.md`:
   ```
   ## All Splits Complete
   Finished  : [ISO timestamp]
   Total time: [duration]
   Splits    : [N completed / N total]
   Commits   : [N]

   Summary:
   [3-6 sentences: what was built, key decisions made during execution, anything flagged or skipped]
   ```

2. Proceed to the Post-Execution Build Gate below. Phase 7 begins only after the build passes.

### Post-Execution Build Gate

After ALL splits are complete and before Deep Review:

The build gate runs on the LAST split's branch (`<task-branch>/split-N`), which contains all changes from all prior splits via chaining.

1. Run FULL project build. Auto-detect from repo:
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
2. If build **FAILS**:
   - Write build errors to `forge-coordination.md`
   - Present to user with options:
     a. Fix build errors (re-enter RALPH loop for affected files)
     b. Revert to last good checkpoint
     c. Stop and fix manually
3. If build **PASSES** → proceed to Phase 7 (Deep Review)

---

## Phase 7 — Deep Review Loop

All code changes are complete and the build has passed. Deep review runs locally against the branch diff.

Update `forge-state.md`: set `phase: deep-review`.

### Step 1 — Run Deep Review

Deep review runs locally against the current branch's diff from the base branch (specified by the user in Phase 5, stored as `$BASE_BRANCH`).

Generate the full diff:

```powershell
# PowerShell
git diff "origin/$BASE_BRANCH..HEAD" | Out-File -Encoding utf8NoBOM forge-deep-review-diff.patch
# Fallback for PowerShell 5.1: [IO.File]::WriteAllText("forge-deep-review-diff.patch", (git diff "origin/$BASE_BRANCH..HEAD" | Out-String), [Text.UTF8Encoding]::new($false))
```
```bash
# Bash
git diff origin/$BASE_BRANCH..HEAD > forge-deep-review-diff.patch
```

**Large diff handling**: If the diff exceeds ~80,000 characters (roughly half the context window for most models):
1. Split the diff into per-file chunks: `git diff origin/$BASE_BRANCH..HEAD -- <file>` for each changed file
2. Group files into batches that fit within context (~80k chars each)
3. Dispatch one set of 3 deep-review agents PER BATCH (architect/advocate/skeptic)
4. After all batches complete, dispatch a final synthesis round where each perspective receives a summary of their own per-batch findings and produces a unified verdict

If the diff fits within context, dispatch all three agents with the full diff as below.

Dispatch three parallel agents using the task tool:

```
task(agent_type="deep-review/architect", prompt="Review this diff for direction and design soundness:\n\n[contents of forge-deep-review-diff.patch]")
task(agent_type="deep-review/advocate", prompt="Defend this implementation — find strengths and justify decisions:\n\n[contents of forge-deep-review-diff.patch]")
task(agent_type="deep-review/skeptic", prompt="Attack this code — find flaws, risks, and weaknesses:\n\n[contents of forge-deep-review-diff.patch]")
```

Wait for all three to complete.

### Step 2 — Evaluate Deep Review Feedback

Write all deep review findings to `forge-coordination.md` under a `## Deep Review — Round [N]` section, attributed by perspective.

Are all three perspectives satisfied with no actionable issues?
- **YES** → proceed to Step 4 (final sign-off)
- **NO**  → proceed to Step 3 (address feedback)

### Step 3 — Address Feedback Loop

For each round of deep review feedback:

1. Triage findings from all three perspectives:
   - Group by file or concern
   - Identify conflicts between perspectives (e.g. Architect wants X, Skeptic flags X as risky) — flag these explicitly and ask user to decide before proceeding

2. Run the forge execution loop (same as Phase 6) against the feedback:
   - Treat each grouped concern as a mini-split
   - One code agent per file affected
   - Dependency-aware, parallel where possible
   - Verifiers run per Phase 6 rules (plan always; architecture + design at mini-split completion for large tasks; skipped for small tasks)
   - Scribe logs each iteration under `## Deep Review Round [N] — Iteration [I]`

3. Commit and push each fix:
   - **Stage only specific files** (`git add <files>`)
   - **Fetch and rebase before push**:
     ```bash
     git fetch origin
     ```

     For **single-split tasks** (only split-1) — rebase against the base branch:
     ```bash
     git rebase origin/$BASE_BRANCH
     ```

     For **multi-split tasks** — rebase against the parent of the current (last) split branch:
     ```bash
     # If on split-N where N > 1, rebase against the previous split
     git rebase origin/<task-branch>/split-<N-1>
     ```
   - Meaningful commit message referencing the deep review concern addressed
   - Follow host environment's commit trailer policy

4. Increment `deep-review-round` in `forge-state.md` and update `## Deep Review` last result.

5. Re-run deep review (regenerate diff and dispatch agents) — go back to Step 2.

Hard cap: max 5 deep review rounds. If cap is reached:
- Pause and present remaining open issues to user
- Ask how to proceed — do not auto-resolve or mark PR ready

### Step 4 — Final Sign-Off

Once all three deep review perspectives are satisfied:

1. Scribe appends to `forge-task-log.md`:
   ```
   ## Deep Review Complete
   Rounds    : [N]
   Finished  : [ISO timestamp]

   All perspectives satisfied. Branch ready for user to create PR.
   ```

2. Report to user:
   ```
   All done.

   Branches : [list all split branches: <task-branch>/split-1, <task-branch>/split-2, ..., <task-branch>/split-N]
   Commits  : [N total across all splits]
   Log      : [path to forge-task-log.md]

   Deep review passed after [N] round(s).
   Create PRs from each split branch when you're ready for team review.
   Each split branch chains from the previous — review and merge in order.
   ```

3. Offer cleanup:

   List only the forge working files that actually exist in the project folder. For small tasks, `forge-verifier-arch.md`, `forge-verifier-design.md`, `forge-summary-arch.md`, and `forge-summary-design.md` were never created — do not list them.

   ```powershell
   # PowerShell — discover existing forge files
   Get-ChildItem -Filter "forge-*" | ForEach-Object { "- $($_.Name)" }
   ```
   ```bash
   # Bash
   ls forge-* 2>/dev/null | sed 's/^/- /'
   ```

   ```
   Forge created the following working files:
   [dynamically list only files that exist]

   These are in .gitignore and won't be committed.
   Would you like me to delete them? (They're useful for debugging if you keep them.)
   ```

4. Offer checkpoint tag cleanup:

   ```
   Forge created the following checkpoint tags:
   [list all forge-checkpoint--*--split-* tags]

   Would you like me to delete these tags? (They were used for rollback safety during execution.)
   ```

   If user confirms, delete tags locally and remotely:
   ```powershell
   # PowerShell
   git tag -l "forge-checkpoint--*" | ForEach-Object { git tag -d $_; git push origin --delete $_ 2>$null }
   ```
   ```bash
   # Bash
   git tag -l 'forge-checkpoint--*' | xargs -I{} sh -c 'git tag -d {} && git push origin --delete {} 2>/dev/null'
   ```

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
- Always add `forge-*.md` to `.gitignore` at startup
- Always flag conflicting feedback between deep review perspectives and ask user before resolving
- Always cap deep review rounds at 5 — if not satisfied by then, pause and ask user
- Architecture doc is required for LARGE tasks — if not provided, block execution and ask the user. For SMALL tasks (as classified by Crucible), skip architecture and design doc requirements entirely.
- Build gate runs once after all splits complete, before deep review — not during the RALPH loop.
- Deep review runs locally against the branch diff — no PR required.
- Always ask the user for the base branch in Phase 5 — never auto-detect or hardcode main/master.
- Always ask the user if splits are chained or independent in Phase 5 — independent splits should be separate Forge executions.
- Always dispatch agents explicitly using task(agent_type='foundry/<agent-name>') — never use vague instructions.
- Each split gets its own branch (<task-branch>/split-N), chained from the previous split. Never put all splits on one branch.
- Always ask the user for their branch naming preference/prefix in Phase 5 (before execution) — never duplicate the prompt in Phase 6.
- Always check for an existing branch (local → remote) BEFORE creating a new one from a parent — this prevents resume divergence.
- When creating split-N branches, always include `origin/<task-branch>/split-<N-1>` as a final fallback parent for fresh-environment resume.
- Always provide both PowerShell and Bash variants for shell commands. Never use bash-only syntax (e.g., `2>/dev/null`) in PowerShell blocks — use `2>$null` or try/catch instead.
- Always use `Out-File -Encoding utf8NoBOM` (PowerShell 7+) or `[IO.File]::WriteAllText` with `[Text.UTF8Encoding]::new($false)` (PowerShell 5.1) when writing patch/diff files — default `>` produces UTF-16, and `-Encoding UTF8` adds a BOM, both of which break git apply.
- Always slugify branch names in checkpoint tags (replace `/` with `-`) to prevent git ref path conflicts with hierarchical branch names.
- When extending the hard cap, always persist the new value to `hard-cap-iterations` in `forge-state.md` before continuing.
- When the deep review diff exceeds ~80k characters, chunk it into per-file batches and run parallel deep review agents per batch, then synthesize.
- Always offer to clean up checkpoint tags and per-file diff patches at task completion.
