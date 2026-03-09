---
name: crucible
description: Multi-model plan refinery — dispatches tasks to 3 AI models (Opus, Codex, Gemini) in parallel, cross-reviews adaptively until convergence, and outputs Forge-compatible plan.md + design-doc.md. Triggers on "plan this", "crucible", "/crucible", "design this task", "create a plan for", "I need a plan", "refine this plan".
disable-model-invocation: true
---

# Crucible — Multi-Model Plan Refinery

> **Named after the metallurgy concept:** raw materials are refined in a crucible before entering the forge.

Crucible takes a raw task description, dispatches it to 3 AI models in parallel, cross-pollinates their outputs through adaptive convergence rounds, and produces Forge-compatible `plan.md` + `design-doc.md`.

---

## Phase 1 — Task Intake & Resume Detection

### Resume Check

Before asking for input, check if `crucible-state.md` exists in the working directory:

1. Search for `crucible-state.md` in current dir and immediate subdirectories
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
4. Design doc        — (optional) if not provided, Crucible generates one
5. Output directory  — where to write plan.md and design-doc.md (default: current directory)
```

After user responds:

- If existing plan provided → enter **Validation Mode** (Phase 1.5)
- If existing design doc provided but no plan → enter **Generation Mode** with design doc as context
- If no design doc → ask: "No design doc provided. Should I generate one alongside the plan?" (default: Yes)
- Detect platform from `git remote -v` (ADO vs GitHub) for work item fetching
- Record choices in memory

---

## Phase 1.5 — Validation Mode (Existing Documents)

If the user provides an existing `plan.md` and/or `design-doc.md`, Crucible validates them against Forge's requirements **before** deciding whether to refine.

### Plan Validation

Read the provided plan and check every Forge requirement:

| # | Requirement | Check |
|---|------------|-------|
| 1 | Task splits present | Are there named splits with clear goals? |
| 2 | Splits meaningfully scoped | Is each split independently testable? Not too large (>12 files)? |
| 3 | Branch name specified | Is there a `## Branch` section with a valid git branch name? |
| 4 | File-level breakdown per split | Does every split have a files table with exact paths? |
| 5 | File actions specified | Does every file have CREATE or MODIFY action? |
| 6 | File dependencies specified | Are dependencies listed? Are they free of circular references? |
| 7 | Acceptance criteria per split | Are criteria specific and testable (not vague)? |
| 8 | Test strategy per split | Is there a concrete test approach? |
| 9 | Splits ordered correctly | Do later splits only depend on earlier ones? |
| 10 | No contradictions between splits | Do splits reference consistent file paths and interfaces? |

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

**Strict checks**: Items 1, 3, 4, 5, 6 are blocking — plan/design CANNOT proceed to Forge without these. Items 2 is a warning (non-blocking but flagged).

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

For each input provided:

- If file path → read directly
- If directory → search for `*.md`, `*.txt`, `architecture*`, `design*`, `plan*`, `spec*`
- If URL → fetch content
- If work item ID → fetch via ADO MCP tools (if ADO detected) or GitHub issues
- If plain text → use directly

Read existing codebase structure:

- Discover project files using cross-platform commands:
  - **Windows (PowerShell)**: `Get-ChildItem -Recurse -Include *.ts,*.cs,*.py,*.go,*.java,*.rs | Select-Object -First 100 FullName`
  - **Unix/Mac**: `find . -type f \( -name "*.ts" -o -name "*.cs" -o -name "*.py" -o -name "*.go" -o -name "*.java" -o -name "*.rs" \) | head -100`
  - Or use the `glob` tool if available: `**/*.{ts,cs,py,go,java,rs}`
- Read key config files (`package.json`, `tsconfig.json`, `.csproj`, `Cargo.toml`, `go.mod`, etc.)
- Understand the project structure and tech stack

Assemble the **context packet** — a single markdown document that all 3 models will receive identically:

```markdown
# Crucible Context Packet

## Task Description
[full task text]

## Architecture Context
[architecture doc content or "Not provided"]

## Existing Design Doc
[design doc content or "Not provided — Crucible will generate"]

## Codebase Structure
[file tree summary, tech stack, key patterns observed]

## Instructions
You are one of 3 models participating in a Crucible refinement process.
Your job is to produce a Forge-compatible plan (and design doc if indicated).
Be specific — include exact file paths, concrete types, real method signatures.
Do not be generic or hand-wavy.
```

---

## Phase 3 — Fleet Dispatch (Round 1: Independent Planning)

Dispatch 3 parallel sub-agents using the task tool:

- **Agent 1**: model `claude-opus-4.6`
- **Agent 2**: model `gpt-5.1-codex-max`
- **Agent 3**: model `gemini-3-pro-preview` (NOTE: use `mode="sync"` for Gemini — it fails on background dispatch)

Each model dispatches TWO agents from the foundry plugin:

1. **Plan Drafter** — `task(agent_type="foundry/plan-drafter", prompt=<context packet + round instructions>, model=<model>)`
2. **Design Drafter** (if generating) — `task(agent_type="foundry/design-drafter", prompt=<context packet + round instructions>, model=<model>)`

The agent instructions in `agents/plan-drafter.md` and `agents/design-drafter.md` define the exact output templates and quality requirements. Do NOT duplicate those templates here — the agents are the single source of truth.

For each model (Opus, Codex, Gemini), dispatch both agents with the context packet as the prompt. The context packet should include:
- The full context assembled in Phase 2
- Round number (1 for initial)
- Whether design doc generation is needed

Collect outputs from all 6 agents (3 plan-drafters + 3 design-drafters).

Store each model's output in the output directory:

- `crucible-round-1-opus.md`
- `crucible-round-1-codex.md`
- `crucible-round-1-gemini.md`

Write initial `crucible-state.md`:

```markdown
# Crucible State
Task      : [summary]
Round     : 1 / 10
Status    : round-1-complete
Models    : [opus, codex, gemini]
Generate  : [plan-only | plan-and-design]
Output Dir: [path]

## Round History
| Round | Opus | Codex | Gemini | Converged |
|-------|------|-------|--------|-----------|
| 1     | done | done  | done   | N/A       |
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
4. Produce your REVISED plan.md (and design-doc.md if applicable)
5. At the END, add a convergence assessment:

### Convergence Assessment
- CONVERGED: [yes/no]
- If no, list remaining disagreements:
  1. [specific disagreement]
  2. [specific disagreement]
- Key changes from your previous version:
  1. [what you changed and why]
```

3. Store outputs: `crucible-round-N-opus.md`, `crucible-round-N-codex.md`, `crucible-round-N-gemini.md`

4. **Convergence Detection**: After all 3 complete, check:
   - Did all 3 say "CONVERGED: yes"? → **Exit loop, proceed to Phase 5**
   - Did 2/3 say converged? → Run ONE more round for the holdout
   - Did none converge? → Continue to next round

### Structural Convergence Verification

Do NOT rely solely on models self-reporting "CONVERGED: yes". After collecting round N outputs, the orchestrator performs independent structural checks:

1. **Branch name** — exact match across all 3 models
2. **Split count** — same number of splits
3. **Split names** — all 3 models describe the same work in each split (same intent, same scope — wording differences are fine)
4. **File lists per split** — all 3 models list the same files for each split (minor differences of 1-2 files are acceptable if the extra files are reasonable)
5. **Dependency graph** — all 3 models agree on which files depend on which other files (same edges, same direction — ordering of independent files may differ)

**Convergence criteria** (ALL must be true):
- ≥ 2 of 3 models self-report CONVERGED: yes
- Structural checks 1-5 pass
- No model introduced NEW splits or removed existing ones from round N-1

If models say converged but structural checks fail → override: NOT converged, continue looping.
If structural checks pass but models say not converged → accept convergence (models are being overly cautious).

5. Update `crucible-state.md` with round results

6. **Hard cap at round 10**: If not converged after 10 rounds, present:

```
Crucible has not fully converged after 10 rounds.
Remaining disagreements:
[list each specific point where models still disagree]

Options:
1. Accept best-effort merged output (recommended)
2. Pick one model's output as winner
3. Extend by 5 more rounds
4. Abort
```

**IMPORTANT for Gemini**: Always dispatch Gemini with `mode="sync"` — it consistently fails on background/async dispatch. Opus and Codex can use background mode.

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
- [ ] File dependencies specified and free of circular references
- [ ] Acceptance criteria per split (specific, testable)
- [ ] Test strategy per split
- [ ] No contradictions between splits
- [ ] Splits are ordered respecting inter-split dependencies

If any check fails → dispatch one targeted model round to fix ONLY the failing checks.

### design-doc.md Validation (Forge Requirements)

Run these checks — ALL must pass:

- [ ] Problem statement present and clear
- [ ] Proposed solution approach described
- [ ] New types/interfaces/classes listed with signatures
- [ ] API contracts and data model changes specified
- [ ] Error handling strategy defined (concrete, not vague)
- [ ] Alternatives considered with rejection reasons

If any check fails → dispatch one targeted model round to fix ONLY the failing checks.

### Cross-Check (same as Forge Phase 3)

- Does the plan align with the design doc?
- Does the design respect architecture constraints (if arch doc provided)?
- Are there contradictions between plan and design?
- Are file dependencies free of circular references across ALL splits?

---

## Phase 6 — Output & Cleanup

Write final files to the output directory:

```
<output-dir>/
  plan.md
  design-doc.md     ← only if generated by Crucible
```

### Safe Cleanup (two-phase)

1. **Write final files first** — write `plan.md` and `design-doc.md` to the output directory
2. **Verify outputs exist** — confirm both files exist and are non-empty (`Test-Path` / `stat`)
3. **THEN delete intermediates** — only after verification:
   - `crucible-round-*.md` (all round outputs)
   - `crucible-state.md` (convergence tracker)
4. If verification fails — DO NOT delete intermediates. Warn the user:
   ```
   ⚠️ Output verification failed. Intermediate files preserved for recovery.
   ```

**Rule 8 gate**: Do not delete ANY files until the user has chosen an option (Hand off / Push / Done).

Present completion summary:

```
╔══════════════════════════════════════╗
║         Crucible Complete            ║
╠══════════════════════════════════════╣
║ Rounds    : [N] (converged at [N])  ║
║ Models    : Opus, Codex, Gemini     ║
║                                      ║
║ Output:                              ║
║   → plan.md       [X splits, Y files]║
║   → design-doc.md [generated/provided]║
╠══════════════════════════════════════╣
║ Options:                             ║
║ 1. Hand off to Forge (/forge)        ║
║ 2. Push to GitHub repo               ║
║ 3. Done — review manually            ║
╚══════════════════════════════════════╝
```

---

## Phase 7 — Optional Actions

### Option 1: Hand off to Forge

- Tell the user to run `/forge` and point it at the output directory
- Forge will pick up `plan.md` and `design-doc.md` automatically

### Option 2: Push to GitHub

- Check `gh auth status` — if not logged in, guide through `gh auth login`
- Ask: "Create new repo or push to existing?"
  - New: `gh repo create <name> --private --source=. --push`
  - Existing: `git init && git add plan.md design-doc.md && git commit && git remote add && git push`

### Option 3: Done

- Confirm files are written and exit

---

## Rules

1. **Never skip phases** — even if the task seems simple
2. **Never fabricate convergence** — if models disagree, keep looping (up to cap)
3. **Identical context packets** — all 3 models must receive byte-identical input each round
4. **Forge compatibility is non-negotiable** — `plan.md` and `design-doc.md` MUST pass Forge's Phase 3 parser
5. **Platform awareness** — detect ADO vs GitHub from git remote, use appropriate MCP tools
6. **Clean up intermediates** — only `plan.md` and `design-doc.md` survive in the output directory
7. **Gemini sync mode** — always dispatch Gemini with `mode="sync"`, never background
8. **User approval before cleanup** — don't delete intermediate files until the user has chosen an option in Phase 6
9. **State persistence** — write `crucible-state.md` after every round so the process can resume
10. **Token cost awareness** — in rounds 2+, only pass the PREVIOUS round's outputs (not all historical rounds) to keep context manageable
