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
3. If valid, present to the user:
   ```
   Found existing Forge state:
     Branch    : <branch>
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
Skip anything that doesn't apply.

1. Architecture doc   — global/shared document (file or folder path)
2. Design doc         — task-specific (file or folder path)
3. Plan               — task-specific; often same directory as design doc
4. Product spec       — optional
5. Task description   — brief summary or work item ID
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
- Is the file dependency graph acyclic? (Validate DAG — flag cycles as blocking errors)

### Plan-Specific Validation (required for execution)

- [ ] Task splits are present and meaningfully scoped
  - If missing → `[MISSING]` warn, offer to create splits yourself or ask user to add them
- [ ] Branch name is specified
  - If missing → `[MISSING]` warn, offer to define one yourself or ask user
- [ ] File-level breakdown exists per split
  - If missing → `[MISSING]` warn — execution cannot safely spawn per-file agents without this
- [ ] File dependencies are specified per split
  - If missing → `[WARN]` execution will assume all files in a split are independent

### Generate Document Summaries

**[FIX #4: Token Cost Optimization]**

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
- Architecture : [path or NOT PROVIDED]
- Design Doc   : [path or NOT PROVIDED]
- Plan         : [path or NOT PROVIDED]
- Product Spec : [path or NOT PROVIDED / NOT REQUIRED]

### Task Summary
[2-4 sentences: what the task is, proposed approach, scope]

### What Looks Good
- [Each doc found and appears complete]
- [Specific strengths, e.g. "Plan has file-level breakdown with dependencies"]

### Gaps and Warnings
- [MISSING]     Architecture doc not provided — proceeding without system-level constraints
- [MISSING]     No branch name in plan
- [MISSING]     No file-level breakdown in plan — cannot safely spawn per-file agents
- [INCOMPLETE]  Design doc missing error handling strategy
- [WARN]        No file dependencies specified — will assume all files in each split are independent
- [MISMATCH]    Plan references module X but design doc does not mention it
- [NOTE]        Product spec marks feature Y as out of scope — confirm before implementing

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

Present a concise execution preview:

```
## Ready to Forge

Branch   : [branch name from plan]
Splits   : [N splits — list them with their file counts]
Strategy : One code agent per file, parallel within dependency constraints
           Verifiers (plan every iteration; architecture + design at split completion)
           Build gate before every commit
           Scribe maintains task log at [project folder path]

Shall I proceed?
```

Wait for explicit user confirmation. Do not touch any file until confirmed.

---

## Phase 6 — Execution

### Setup

Before starting the loop:

#### 1. Environment Preflight Checks

**[FIX #5: Git Safety]**

```bash
# Verify clean working tree
git status --porcelain
# If output is non-empty → STOP, ask user to stash or commit

# Verify remote is reachable
git ls-remote --exit-code origin
# If fails → STOP, ask user to check network/auth

# Verify authentication works for push
git push --dry-run origin HEAD 2>&1
# If fails → STOP, ask user to fix credentials
```

If any preflight check fails, do NOT proceed. Report the failure and wait for the user.

#### 2. Platform Detection

**[FIX #2: Platform Detection]**

Detect the hosting platform from git remotes:

```bash
git remote -v
```

Parse the output:
- If URL contains `dev.azure.com` or `visualstudio.com` → **ADO platform**. Use `ado-repo_create_pull_request` for PR operations.
- If URL contains `github.com` → **GitHub platform**. Use `github-mcp-server` tools for PR operations.
- If neither detected → **Unknown platform**. Skip PR creation, notify user: "Could not detect platform — PR will not be created automatically. You can create one manually after execution."

Store the detected platform in `forge-state.md` under `## Platform`.

#### 3. Forge File Hygiene

**[FIX #10: Forge File Hygiene]**

Ensure forge artifacts don't pollute version control:

```bash
# Check if forge-*.md is already in .gitignore
grep -q 'forge-\*\.md' .gitignore 2>/dev/null
if [ $? -ne 0 ]; then
  echo '' >> .gitignore
  echo '# Forge orchestration artifacts' >> .gitignore
  echo 'forge-*.md' >> .gitignore
fi
```

If `.gitignore` doesn't exist, create it with the forge pattern. If modifying `.gitignore`, stage and commit it immediately with message: `chore: add forge artifacts to .gitignore`.

#### 4. Create Coordination Files

Create the coordination directory in the project folder (same folder as plan/design doc):

```
<project-folder>/
├── forge-state.md               ← orchestrator loop state
├── forge-coordination.md        ← consolidated verifier feedback
├── forge-verifier-plan.md       ← plan verifier output (isolated)
├── forge-verifier-arch.md       ← architecture verifier output (isolated)
├── forge-verifier-design.md     ← design verifier output (isolated)
├── forge-task-log.md            ← scribe log
├── forge-summary-arch.md        ← architecture summary for agents
├── forge-summary-design.md      ← design summary for agents
└── forge-summary-plan.md        ← plan summary for agents
```

#### 5. Initialize Task Log

```markdown
# Forge Task Log
Task: [task description]
Branch: [branch name]
Started: [ISO timestamp]
Splits: [N]
Platform: [ADO / GitHub / Unknown]
---
```

#### 6. Checkout or Create Branch

```bash
git fetch origin
git checkout -b <branch-name> origin/main || git checkout <branch-name>
```

#### 7. Create Draft PR

Based on detected platform:

**ADO:**
```
ado-repo_create_pull_request:
  isDraft: true
  targetRefName: "refs/heads/main"
  sourceRefName: "refs/heads/<branch-name>"
  title: <concise task description>
  description: <task summary, list of splits>
```

**GitHub:**
```
github-mcp-server tools:
  Create PR as draft with equivalent parameters
```

**Unknown platform:**
Skip PR creation. Log: "PR creation skipped — platform not detected."

Write the PR number and URL to `forge-state.md` under `## Deep Review`.
**Do not publish or mark as ready — draft only.**

#### 8. Initialize State File

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
platform: [ADO / GitHub / Unknown]
---

## Platform
- Type     : [ADO / GitHub / Unknown]
- Remote   : [remote URL]
- PR Tools : [ado-repo / github-mcp-server / none]

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
- Architecture: —
- Design      : —

## Deep Review
- Round       : 0
- PR URL      : [URL or "not created"]
- PR Number   : [number or "N/A"]
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
> You allocate the array with the required backing specifications, give it a goal, then loop the goal.
> Each iteration feeds the AI context from its previous work — git history, modified files, verifier feedback —
> creating a self-referential feedback loop that lets the system improve its own output over time.
>
> `forge-state.md` is the RALPH loop's **persistent memory**. It carries the full context of where
> the loop is (split, iteration, dependency status, agent outcomes, verifier verdicts) across iterations.
> Without it, every spin of the wheel starts blind. With it, each iteration is informed by everything
> that came before — failures, corrections, approvals. This is how the loop self-corrects.
>
> The loop runs **per split** — each split is an independent turn on the pottery wheel.
> If something isn't right, it goes back on the wheel. When all verifiers approve and the build passes,
> the clay is fired (committed). Then the next split goes on the wheel.

Repeat for each split in order:

```
SPLIT START (put the clay on the wheel)
  Read plan split definition from disk
  Build dependency graph: file → dependencies[]
  Reset agent status map and iteration count
  Write split header to forge-task-log.md

  RALPH LOOP (spin the wheel until the clay is right):
    1. RESOLVE UNBLOCKED FILES
       Files whose dependencies are all in done status are unblocked.
       On first iteration, files with no dependencies are unblocked.

    2. SPAWN CODE AGENTS (Task tool, parallel, one per unblocked file)
       Each code agent receives:
         - The specific file it owns
         - Document SUMMARIES (not full docs) — forge-summary-arch.md, forge-summary-design.md, forge-summary-plan.md
         - The task split description
         - Contents of forge-coordination.md (verifier feedback from last iteration)
         - Instruction: implement only what is needed for this file in this split
         - Instruction: do not commit
         - Instruction: do not modify any file other than your assigned file
         - Path to code-agent.md instructions

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

       **[FIX #3: Verifier Write Isolation]**

       Each verifier writes to its OWN isolated file — never to the shared coordination file.

       **[FIX #4: Token Cost — Selective Verifier Runs]**

       - **Plan Verifier** → runs EVERY iteration
         - Input: plan doc, per-file diffs, file list, split description
         - Output: forge-verifier-plan.md
         - Path to plan-verifier.md instructions

       - **Architecture Verifier** → runs only at SPLIT COMPLETION (all files done)
         - Input: architecture doc, per-file diffs, file list, split description
         - Output: forge-verifier-arch.md
         - Path to architecture-verifier.md instructions

       - **Design Verifier** → runs only at SPLIT COMPLETION (all files done)
         - Input: design doc, per-file diffs, file list, split description
         - Output: forge-verifier-design.md
         - Path to design-verifier.md instructions

       Generate per-file diffs (not full-repo diff):
       ```bash
       git diff HEAD -- <file1> > /tmp/forge-diff-file1.patch
       git diff HEAD -- <file2> > /tmp/forge-diff-file2.patch
       ```

       Wait for ALL active verifiers to complete.

       **Consolidation**: After all verifiers finish, read each `forge-verifier-*.md` file and consolidate into `forge-coordination.md`:
       ```markdown
       ## Consolidated Verifier Feedback — Split [N] Iteration [I]

       ### Plan Verifier
       [copied from forge-verifier-plan.md]

       ### Architecture Verifier
       [copied from forge-verifier-arch.md, or "SKIPPED — runs at split completion"]

       ### Design Verifier
       [copied from forge-verifier-design.md, or "SKIPPED — runs at split completion"]
       ```

    6. SCRIBE (conditional, after verifiers)

       **[FIX #4: Token Cost — Conditional Scribe]**

       Invoke Scribe:
         - If ANY verifier returned ISSUES FOUND
         - If this is the final approved iteration of a split
         - If this is a deep review iteration

       Skip Scribe on clean mid-split iterations where all active verifiers approve
       and there's nothing notable to record.

       When invoked, Scribe reads:
         - forge-coordination.md (consolidated verifier outputs)
         - Per-file diffs of this iteration
         - Agent status map

       Scribe appends one log entry to forge-task-log.md.

    7. EVALUATE ITERATION OUTCOME
       All active verifiers APPROVED?
         YES → proceed to BUILD GATE
         NO  → iteration count < hard cap?
                 YES → clear APPROVED sections in forge-coordination.md,
                        keep ISSUES sections → next iteration
                 NO  → HARD CAP REACHED (see below)

    8. BUILD GATE

       **[FIX #1: Build/Test Gate]**

       Before committing, run the project's build command. Auto-detect from repo:

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
       - Write compiler/test errors to `forge-coordination.md` under `## Build Errors`
       - Treat as ISSUES FOUND — loop back to next iteration
       - This counts toward the hard cap

       If build **PASSES**:
       - Proceed to COMMIT AND PUSH

    9. COMMIT AND PUSH

       **[FIX #7: Specific File Staging]**

       Stage ONLY the specific files assigned to this split's agents:
       ```bash
       git add <file1> <file2> <file3>
       ```

       Do NOT use `git add .` or `git add -A`. Only stage files that code agents were assigned to modify.

       **[FIX #5: Git Safety — Pre-push Rebase]**

       Before pushing:
       ```bash
       git fetch origin
       git rebase origin/<target-branch>
       ```

       If rebase **fails** (conflicts):
       - `git rebase --abort`
       - PAUSE execution
       - Report to user: "Rebase conflict detected. Please resolve manually and run 'continue'."
       - Do NOT force-push or auto-resolve

       **[FIX #6: Co-authored-by Flexibility]**

       Write a commit message that:
       - Describes what was implemented (not "iteration N" or "split N")
       - References the specific behavior or feature delivered
       - Is concise and meaningful
       - Follows the host environment's commit trailer policy. If the environment requires `Co-authored-by` or other trailers, include them. If no policy exists, omit agent signatures.

       Commit and push:
       ```bash
       git commit -m "<meaningful message>"
       git push origin <branch-name>
       ```

       The draft PR updates automatically with each push.

SPLIT END → clay is fired, move to next split (next piece on the wheel)
```

> **Watching the loop is where you learn.** When you see a failure domain — a verifier
> that keeps rejecting, a build that won't pass, a dependency that blocks everything —
> put on your engineering hat and resolve the problem so it never happens again.
> The RALPH loop is not just an execution pattern; it's a learning pattern.

### Hard Cap Behavior (Knowing When to Stop the Wheel)

**[FIX #9: Hard Cap Enforcement]**

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

---

### Completion

After all splits are done:

1. Scribe writes a final summary to `forge-task-log.md`:
   ```
   ## Task Complete
   Finished  : [ISO timestamp]
   Total time: [duration]
   Splits    : [N completed / N total]
   Commits   : [N]
   Platform  : [ADO / GitHub / Unknown]

   Summary:
   [3-6 sentences: what was built, key decisions made during execution, anything flagged or skipped]
   ```

2. Proceed immediately to Phase 7.

---

## Phase 7 — Deep Review Loop

The draft PR was created at the start of execution and has been receiving every commit pushed during Phase 6. All code changes are now complete on the remote.

Update `forge-state.md`: set `phase: deep-review`.

If platform is "Unknown" (no PR was created), skip Deep Review and go to Completion with a note: "Deep review skipped — no PR available. Consider running `/deep-review` manually on the branch."

### Step 1 — Run Deep Review

Read the PR number from `forge-state.md` → `## Deep Review → PR Number`.

Invoke `/deep-review <PR-Number>` — deep-review will diff the PR against main directly.

Deep review runs three parallel perspectives:
- **Architect** — direction and design soundness
- **Advocate** — defense of the implementation
- **Skeptic** — attack mindset, finds flaws

Wait for all three perspectives to complete.

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
   - Plan Verifier checks that fixes align with the original plan
   - Build gate runs before every commit
   - Scribe logs each iteration under `## Deep Review Round [N] — Iteration [I]`

3. Commit and push each fix:
   - **Stage only specific files** (`git add <files>`)
   - **Fetch and rebase before push** (`git fetch origin && git rebase origin/<target>`)
   - Meaningful commit message referencing the deep review concern addressed
   - Follow host environment's commit trailer policy

4. Increment `deep-review-round` in `forge-state.md` and update `## Deep Review` last result.

5. Re-run `/deep-review <PR-Number>` — go back to Step 2.

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

   All perspectives satisfied. Draft PR ready for user review.
   ```

2. Report to user:
   ```
   All done.

   Draft PR : [PR URL]
   Branch   : [branch name]
   Commits  : [N total]
   Log      : [path to forge-task-log.md]

   Deep review passed after [N] round(s).
   The PR is still a draft — mark it ready for review when you're happy.
   ```

3. **[FIX #10: Forge File Hygiene — Cleanup Offer]**

   ```
   Forge created the following working files:
   - forge-state.md
   - forge-coordination.md
   - forge-verifier-plan.md
   - forge-verifier-arch.md
   - forge-verifier-design.md
   - forge-task-log.md
   - forge-summary-*.md

   These are in .gitignore and won't be committed.
   Would you like me to delete them? (They're useful for debugging if you keep them.)
   ```

Do not mark the PR as ready. Do not merge. Hand off to the user.

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
- Always create the draft PR at the start of execution (Phase 6 setup), not at the end
- Always push immediately after every commit
- Always fetch and rebase before push — never push without checking for drift
- On rebase conflict, always pause and ask user — never force-push or auto-resolve
- Always stage only specific assigned files — never use `git add .` or `git add -A`
- Always run the build gate before committing — never commit code that doesn't compile
- Always detect platform from git remotes — never hardcode ADO or GitHub
- Each verifier writes to its own file — never have multiple verifiers write to the same file
- Pass document summaries to code agents — never send full documents to every agent
- Follow the host environment's commit trailer policy — never impose or ban specific trailers
- Always add `forge-*.md` to `.gitignore` at startup
- Never publish or mark the draft PR as ready — that is always the user's decision
- Never merge — forge only builds and reviews, never merges
- Always flag conflicting feedback between deep review perspectives and ask user before resolving
- Always cap deep review rounds at 5 — if not satisfied by then, pause and ask user
