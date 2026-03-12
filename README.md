# Foundry -- AI-Powered Task Lifecycle for Copilot CLI

> Plan with 3 AI models. Execute with coordinated agents. Ship with confidence.

> [!CAUTION]
> **AI can make mistakes.** Foundry is a powerful tool, but it is still just a tool. It is your responsibility to cross-check all outputs -- code, plans, and design docs. No matter how smart the orchestration gets, it still needs human intervention. **Please always verify your work before merging.**

> [!IMPORTANT]
> **Copilot CLI only.** Foundry requires multi-model orchestration (dispatching parallel agents to Claude, Codex, and Gemini) which is only available in [GitHub Copilot CLI](https://docs.github.com/copilot/concepts/agents/about-copilot-cli). It does **not** work with Claude Code, Claude Desktop, or other Claude interfaces -- those environments only have access to a single model and cannot run the cross-model review loops that Crucible and Forge depend on.

Foundry turns a task description into committed, pushed code through two skills: **Crucible** plans it, **Forge** builds it.

## What You Get

| Skill | What It Does |
|-------|-------------|
| **Crucible** | Takes your task -> 3 AI models independently plan it -> cross-review each other -> converge on a single plan |
| **Forge** | Takes the plan -> writes code file-by-file -> verifies against plan/design/architecture -> commits and pushes |

You go from "here's my work item" to "here's a branch with reviewed code" without writing a single line yourself.

---

## Installation

### Prerequisites
- [Copilot CLI](https://docs.github.com/copilot/concepts/agents/about-copilot-cli) installed
- **deep-review** plugin (used by Forge's final review step):
  ```
  /plugin marketplace add agency-microsoft/playground
  /plugin install deep-review@agency-playground
  ```

### Install Foundry

```
/plugin install odsp-microsoft/foundry
```

To update to the latest version:
```
/plugin update foundry
```

Verify it's installed:
```
/skills
# Should show: foundry:crucible, foundry:forge
```

---

## Quick Start

### Crucible -- Plan a Task

Just describe what you need. Crucible figures out the rest.

```
I have this work item: "TASK 12345: Standardize enum string conversion
across billing policy code." Analyze and plan this task.
```

Crucible will:
1. Ask you for any architecture docs or context (optional)
2. Ask for execution config (base branch, branch prefix, split relationship)
3. Assess complexity -- is this a quick fix or a big project?
4. Dispatch 3 AI models to independently create a plan
5. Have them cross-review each other until they agree
6. Output a final `plan.md` (and `design-doc.md` if the task is large)

### Forge -- Execute the Plan

Once you have a plan, hand it to Forge:

```
Forge this. Plan is at ./plan.md, architecture doc is at ./docs/architecture.md
```

Forge will:
1. Read and validate the plan
2. Read branch config from the plan (set by Crucible)
3. Write code file-by-file using dedicated code agents
4. Verify every iteration against the plan (and design/architecture for large tasks)
5. Run adversarial deep review and pass the build gate
6. Commit and push

---

## Example Prompts

### Crucible Examples

**From a work item (simplest):**
```
I got this work item -- "TASK 67890: Add retry logic to the blob upload
pipeline." I don't have much info, but analyze and plan this.
```

**With context:**
```
Plan this task: We need to migrate our authentication from cookie-based
to JWT tokens across the API layer. Architecture doc is at ./docs/auth-architecture.md
```

**With an existing plan to validate:**
```
I already have a plan.md -- can you validate it meets Forge's requirements
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
Here's the task -- plan is at ./output/plan.md, design doc is at
./output/design-doc.md, and architecture is at ./docs/architecture.md
```

**Small task (no design doc):**
```
Forge this. Plan is at ./plan.md -- it's a small task, no design doc needed.
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
                    |
            +-------V-------+
            |   CRUCIBLE     |
            |                |
            |  1. Intake     |  <- You provide task + optional docs
            |  2. Assess     |  <- AI decides: small or large? single or multi-split?
            |  3. Config     |  <- Base branch, naming, split relationship
            |  4. Fleet      |  <- 3 models plan independently
            |  5. Converge   |  <- Models cross-review until agreement
            |  6. Output     |  <- plan.md (with execution config) + design-doc.md (if large)
            +-------+-------+
                    |
                    V
You: "Forge this. Plan is at ./plan.md"
                    |
            +-------V-------+
            |    FORGE       |
            |                |
            |  1. Intake     |  <- Paths to plan (+ docs for large tasks)
            |  2. Read       |  <- Locates & reads all documents
            |  3. Analyze    |  <- Generates summaries, checks complexity
            |  4. Readiness  |  <- Validates plan fields, shows report
            |  5. Confirm    |  <- Shows preview, you say "go" (last prompt)
            |  6. Execute    |  <- Code → verify → iterate (headless, RALPH loop)
            |  7. Wrap up    |  <- Summary, push, cleanup
            +---------------+
```

---

## What Crucible Asks You

During planning, Crucible collects everything Forge needs so Forge can run headless:

| Prompt | When | Your Options |
|--------|------|-------------|
| Task description | Always | Text, file path, URL, or work item ID |
| Architecture doc | Always (optional) | File path or "none" |
| Existing plan/design doc | Always (optional) | File path or "none" |
| Complexity confirmation | After assessment | Agree with AI recommendation or override |
| Base branch | Always | `main`, `master`, `develop`, `build/main/latest`, etc. |
| Branch prefix | Always | `user/<alias>/<name>`, `feature/<name>`, custom |
| Split relationship | Multi-split tasks | Chained (builds on previous) or independent |
| Split strategy | After assessment | Single branch (small tasks) or multi-split |

> **Everything goes into the plan.** Crucible writes your branch/split preferences into `plan.md` so Forge reads them automatically.

## What Forge Asks You

Forge is designed to be **headless** -- it reads everything from the plan and only prompts once:

| Prompt | When | Purpose |
|--------|------|---------|
| Plan location | Start (Phase 1, step 1) | Path to plan.md + task description |
| Remaining docs | After reading plan (Phase 1, step 2) | Architecture + design doc — only asked for large tasks |
| "Shall I proceed?" | After readiness check (Phase 5) | Final go/no-go before execution |

That's it. After you say "go", Forge runs to completion without interruption -- writing code, verifying, committing, pushing, and reviewing autonomously.

> **Fallback:** If your plan wasn't made by Crucible and is missing `## Execution Config`, Forge will prompt for base branch and prefix as a one-time fallback.

---

## Small vs Large Tasks

Crucible automatically assesses whether your task is small or large and adjusts the workflow:

| | Small Task | Large Task |
|---|-----------|------------|
| **Examples** | Bug fix, config change, simple feature | New API, multi-component refactor |
| **Plan** | Yes | Yes |
| **Design doc** | Skipped | Yes |
| **Architecture doc** | Not required | Required |
| **Verifiers in Forge** | Plan only | Plan + design + architecture |
| **Typical time** | Faster, less ceremony | Full ceremony |

You always get to confirm or override the AI's recommendation.

---

## Tips for Best Results

1. **Give Crucible context** -- The more you provide (architecture docs, existing code patterns, constraints), the better the plan
2. **Work item IDs work** -- Just paste the work item ID or URL; Crucible fetches the details
3. **Don't skip Crucible for big tasks** -- Going straight to Forge with a hand-written plan works, but Crucible's 3-model convergence catches gaps you'd miss
4. **Use Forge's resume** -- If Copilot CLI crashes mid-execution, just say "resume" and it picks up where it left off
5. **Branch naming** -- Use your team's convention (e.g., `user/<alias>/<feature>/split-N`). For small single-branch tasks, no `/split-N` suffix is added.
6. **Review the plan before forging** -- Crucible outputs `plan.md` -- read it, tweak it if needed, then hand it to Forge

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| "Missing ## Complexity section" | Your plan.md wasn't generated by Crucible. Add a `## Complexity` section with `Classification: small` or `Classification: large` |
| Forge says "NOT READY" | Check the readiness report -- usually a missing document or plan field |
| Deep review step fails | Make sure the `deep-review` plugin is installed: `/plugin marketplace add agency-microsoft/playground` then `/plugin install deep-review@agency-playground` |
| Forge can't find my plan | Provide the full path: `plan is at C:\path\to\plan.md` |
| Want to skip deep review | Not supported -- it's a safety gate. But it's fast (usually 1-2 rounds) |
| Resume shows stale state | Say "start fresh" when prompted -- it archives the old state |

---

## Architecture & Internals

For technical details -- phase breakdowns, agent specs, RALPH loop mechanics, safety features, and version history -- see [ARCHITECTURE.md](ARCHITECTURE.md).

Want to go deeper? The [Foundry Wiki](https://github.com/odsp-microsoft/foundry/wiki) covers the engineering behind the plugin -- how the RALPH loop works, how 3 AI models converge on a plan (harness engineering), agent specifications, safety and recovery mechanisms, and more.

---

## Acknowledgments

Shoutout to **Ian De La Garza** ([@iandelagarza](https://github.com/iandelagarza)) and **Touseef Liaqat** ([@tliaqat](https://github.com/tliaqat)) — everything I've learnt about GitHub Copilot and Claude, and how to efficiently use these tools, I owe to them.

Thanks to the [**deep-review**](https://github.com/agency-microsoft/.github-private/tree/main/plugins/deep-review) team for creating a splendid adversarial review skill — it powers Foundry's fleet review process and has been instrumental in catching subtle bugs across every version.

---

**v1.6.0** - [Version History](ARCHITECTURE.md#version-history)
