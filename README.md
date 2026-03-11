# Foundry — AI-Powered Task Lifecycle for Copilot CLI

> Plan with 3 AI models. Execute with coordinated agents. Ship with confidence.

Foundry turns a task description into committed, pushed code through two skills: **Crucible** plans it, **Forge** builds it.

## What You Get

| Skill | What It Does |
|-------|-------------|
| **Crucible** | Takes your task → 3 AI models independently plan it → cross-review each other → converge on a single plan |
| **Forge** | Takes the plan → writes code file-by-file → verifies against plan/design/architecture → commits and pushes |

You go from "here's my work item" to "here's a branch with reviewed code" without writing a single line yourself.

---

## Installation

### Prerequisites
- [Copilot CLI](https://docs.github.com/copilot/concepts/agents/about-copilot-cli) installed
- **deep-review** plugin (used by Forge's final review step):
  ```
  /plugin marketplace add agency agency-microsoft/.github-private
  /plugin install deep-review@agency
  ```

### Install Foundry
```
/plugin install shshivakumar_microsoft/foundry
```

Or clone locally (works for private repos your team can't access):
```powershell
git clone https://github.com/shshivakumar_microsoft/foundry.git "$HOME\.claude\plugins\local\foundry"
```

Verify it's installed:
```
/skills
# Should show: foundry:crucible, foundry:forge
```

---

## Quick Start

### Crucible — Plan a Task

Just describe what you need. Crucible figures out the rest.

```
I have this work item: "TASK 12345: Standardize enum string conversion
across billing policy code." Analyze and plan this task.
```

Crucible will:
1. Ask you for any architecture docs or context (optional)
2. Assess complexity — is this a quick fix or a big project?
3. Dispatch 3 AI models to independently create a plan
4. Have them cross-review each other until they agree
5. Output a final `plan.md` (and `design-doc.md` if the task is large)

### Forge — Execute the Plan

Once you have a plan, hand it to Forge:

```
Forge this. Plan is at ./plan.md, architecture doc is at ./docs/architecture.md
```

Forge will:
1. Read and validate the plan
2. Ask you for the base branch and naming preference
3. Write code file-by-file using dedicated code agents
4. Verify every iteration against the plan (and design/architecture for large tasks)
5. Commit, push, and run a final adversarial code review

---

## Example Prompts

### Crucible Examples

**From a work item (simplest):**
```
I got this work item — "TASK 67890: Add retry logic to the blob upload
pipeline." I don't have much info, but analyze and plan this.
```

**With context:**
```
Plan this task: We need to migrate our authentication from cookie-based
to JWT tokens across the API layer. Architecture doc is at ./docs/auth-architecture.md
```

**With an existing plan to validate:**
```
I already have a plan.md — can you validate it meets Forge's requirements
and fix any gaps? Plan is at ./plan.md
```

**Refine a design doc:**
```
Design this task: implement a rate limiter for our public API endpoints.
Here's the architecture doc: ./docs/api-architecture.md
```

### Forge Examples

**Standard execution:**
```
Here's the task — plan is at ./output/plan.md, design doc is at
./output/design-doc.md, and architecture is at ./docs/architecture.md
```

**Small task (no design doc):**
```
Forge this. Plan is at ./plan.md — it's a small task, no design doc needed.
```

**Resume an interrupted session:**
```
Resume
```
Forge detects `forge-state.md` from the previous session and picks up where it left off.

---

## The Two-Skill Workflow

Here's what happens end-to-end when you use both skills together:

```
You: "Plan this task: [description]"
                    │
            ┌───────▼───────┐
            │   CRUCIBLE     │
            │                │
            │  1. Intake     │  ← You provide task + optional docs
            │  2. Complexity │  ← AI decides: small or large task?
            │  3. Fleet      │  ← 3 models plan independently
            │  4. Converge   │  ← Models cross-review until agreement
            │  5. Output     │  ← plan.md + design-doc.md (if large)
            └───────┬───────┘
                    │
                    ▼
You: "Forge this. Plan is at ./plan.md"
                    │
            ┌───────▼───────┐
            │    FORGE       │
            │                │
            │  1. Read plan  │  ← Validates all required fields
            │  2. Confirm    │  ← You pick branch, naming, base
            │  3. Code       │  ← Agent writes each file
            │  4. Verify     │  ← Plan/design/arch verifiers check it
            │  5. Iterate    │  ← Fix issues, re-verify (max 10 rounds)
            │  6. Commit     │  ← Push per split
            │  7. Build      │  ← Full build gate after all splits
            │  8. Review     │  ← 3 adversarial reviewers attack the code
            │  9. Done       │  ← Branch pushed, you create the PR
            └───────────────┘
```

---

## What Crucible Asks You

During planning, Crucible will prompt you for:

| Prompt | When | Your Options |
|--------|------|-------------|
| Task description | Always | Text, file path, URL, or work item ID |
| Architecture doc | Always (optional) | File path or "none" |
| Existing plan/design doc | Always (optional) | File path or "none" |
| Output directory | Always | Path (defaults to current dir) |
| Complexity confirmation | After assessment | Agree with AI recommendation or override |

## What Forge Asks You

During execution, Forge will prompt you for:

| Prompt | When | Your Options |
|--------|------|-------------|
| Document locations | Start | Paths to plan.md, design-doc.md, architecture doc |
| Base branch | Before execution | `main`, `master`, `develop`, `build/main/latest`, etc. |
| Branch prefix | Before execution | `user/<alias>/<name>`, `feature/<name>`, custom |
| Split relationship | Multi-split tasks | Chained (builds on previous) or independent |
| Execution approval | After preview | "Yes" to proceed |

---

## Small vs Large Tasks

Crucible automatically assesses whether your task is small or large and adjusts the workflow:

| | Small Task | Large Task |
|---|-----------|------------|
| **Examples** | Bug fix, config change, simple feature | New API, multi-component refactor |
| **Plan** | ✅ Generated | ✅ Generated |
| **Design doc** | ⚡ Skipped | ✅ Generated |
| **Architecture doc** | Not required | Required |
| **Verifiers in Forge** | Plan only | Plan + design + architecture |
| **Typical time** | Faster, less ceremony | Full ceremony |

You always get to confirm or override the AI's recommendation.

---

## Tips for Best Results

1. **Give Crucible context** — The more you provide (architecture docs, existing code patterns, constraints), the better the plan
2. **Work item IDs work** — Just paste the work item ID or URL; Crucible fetches the details
3. **Don't skip Crucible for big tasks** — Going straight to Forge with a hand-written plan works, but Crucible's 3-model convergence catches gaps you'd miss
4. **Use Forge's resume** — If Copilot CLI crashes mid-execution, just say "resume" and it picks up where it left off
5. **Branch naming** — Use your team's convention (e.g., `user/<alias>/<feature>/split-N`)
6. **Review the plan before forging** — Crucible outputs `plan.md` — read it, tweak it if needed, then hand it to Forge

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| "Missing ## Complexity section" | Your plan.md wasn't generated by Crucible. Add a `## Complexity` section with `Classification: small` or `Classification: large` |
| Forge says "NOT READY" | Check the readiness report — usually a missing document or plan field |
| Deep review step fails | Make sure the `deep-review` plugin is installed: `/plugin install deep-review@agency` |
| Forge can't find my plan | Provide the full path: `plan is at C:\path\to\plan.md` |
| Want to skip deep review | Not supported — it's a safety gate. But it's fast (usually 1-2 rounds) |
| Resume shows stale state | Say "start fresh" when prompted — it archives the old state |

---

## Architecture & Internals

For technical details — phase breakdowns, agent specs, RALPH loop mechanics, safety features, and version history — see [ARCHITECTURE.md](ARCHITECTURE.md).

---

**v1.4.3** · [Version History](ARCHITECTURE.md#version-history)
