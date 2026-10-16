---
name: evaluate-idea
description: Set up an idea for evaluation through the kill-gate funnel. Normalizes the idea, initializes .eval/ state, and prompts the user to run gates individually via /run-gate.
---

# Evaluate-Idea — Setup Orchestrator

You set up a business idea for evaluation by normalizing it and initializing the `.eval/` working directory. You do NOT run any gates — that's `/run-gate`'s job.

## Input

The user's idea description is provided as `$ARGUMENTS` — everything after `/evaluate-idea` on the command line. This may be:
- A **number** (`1`, `2`, or `3`) — references an idea from the latest `/find-ideas` run. Read the corresponding file from `.ideas/idea-{N}.md`. If the file doesn't exist, tell the user to run `/find-ideas` first.
- A **file path** to a previous report (e.g., `output/evaluate-idea/2026-04-12-eval-foo.md`). If the argument looks like a file path (starts with `output/`, ends with `.md`, or contains `/`), read the file first. Extract the **final version** of the idea (post-reshape if applicable). Strip all prior-gate context, reshape history, and verdicts. The idea enters the funnel clean.
- A **one-line pitch** ("a tool that helps German landlords file annual tax returns")
- A **paragraph description**
- A **fully-specified idea** with target user, pricing, etc.

If `$ARGUMENTS` is missing or empty, use `AskUserQuestion` to ask for the idea.

## Context about the user

- Solo full-stack developer
- EU-based
- Bootstrapped — no team, no funding
- Looking for viable business ideas that generate revenue — any business model is acceptable
- Prefers lean, bootstrappable ideas over VC-scale moonshots

## Pipeline

### Step 1: Check for existing evaluation

If `.eval/state.md` exists and contains `status: in_progress`:
- Use `AskUserQuestion` to ask: "Found an incomplete evaluation in .eval/. Start fresh or keep the current one?"
  - "Start fresh" → continue to Step 2
  - "Keep current" → tell the user the current evaluation state and suggest `/run-gate <next-gate>`

### Step 2: Normalize the idea

Parse `$ARGUMENTS` and reformat the idea into the structured format below. Where the user left gaps, fill them in with reasonable assumptions — and **explicitly flag each assumption**.

```
### User-Provided Idea: [Name]
**One-liner:** [What it does in one sentence]
**Problem:** [The specific pain point]
**Solution:** [How this product solves it]
**Target user:** [Who pays for this, be specific]
**Revenue model:** [How it makes money]
**Distribution channel (assumed):** [Specific channel where customers already are]
**Estimated user base (assumed):** [Realistic number of addressable users]
**Key differentiator (assumed):** [Why this beats incumbents]

**Assumptions filled in by orchestrator:**
- [Assumption 1]
- [Assumption 2]
```

Do NOT reject the idea at this stage — let the funnel do its job.

### Step 3: Read pipeline config

Read `pipeline.md` from the project root. Extract:
- **Gate list** (ordered list of domain names, e.g., market, product, finance, sales, legal)
- **Reshape budget** (max reshapes before forcing to FLAG)

If `pipeline.md` doesn't exist, use defaults: gates = [market, product, finance, sales, legal], reshape_budget = 2.

### Step 4: Initialize `.eval/` directory

**Delete the entire `.eval/` directory** if it exists — this removes all stale gate files, reshape records, and state from prior runs. Then recreate:

```
.eval/
  idea.md          ← the structured idea from Step 2
  original-idea.md ← copy of idea.md (preserved even through reshapes)
  state.md         ← pipeline state (see format below)
  gates/           ← empty directory (created for /run-gate to write into)
  reshapes/        ← empty directory (created for /run-gate to write into)
```

**`.eval/state.md` format:**

```markdown
---
status: in_progress
current_gate: [first gate from pipeline.md]
current_step: analyst
pass_number: 1
reshape_depth: 0
reshape_budget: [from pipeline.md]
killed: false
killed_at: null
gates_order: [comma-separated gate list from pipeline.md]
---

## Gates
- market: pending
- product: pending
- finance: pending
- sales: pending
- legal: pending

## Reshape History
(none yet)

## Debates
(none yet)
```

### Step 5: Prompt next action

Display the normalized idea and tell the user:

```
Evaluation initialized. Run gates one at a time:

  /run-gate market        ← start here
  /run-gate --auto        ← or run all gates automatically

You can /clear between gates to save context.
```

## Important

- Do NOT run any gates. Do NOT invoke any analyst, contrarian, or advocate agents.
- Do NOT write to `output/` or `rejected-ideas.md`.
- Your only job is to normalize the idea, set up `.eval/`, and hand off to `/run-gate`.
