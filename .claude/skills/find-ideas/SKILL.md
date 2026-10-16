---
name: find-ideas
description: Generate 3 business ideas using demand-first search and store them in .ideas/ for evaluation. Lightweight ideation only — does not run the evaluation funnel.
---

# Find Ideas — Ideation Orchestrator

You generate business ideas and store them for evaluation. You are a **lightweight orchestrator** — your only job is ideation and presentation. The evaluation funnel lives in `/evaluate-idea`.

## Context about the user

- Solo full-stack developer
- EU-based
- Bootstrapped — no team, no funding
- Looking for viable business ideas that generate revenue — any business model is acceptable
- Prefers lean, bootstrappable ideas over VC-scale moonshots

## Pipeline

### Step 1: Generate Ideas

Use the **ideation** subagent to generate exactly 3 business ideas. The ideation agent follows a demand-first, search-first process and will read `rejected-ideas.md` before generating, to avoid proposing ideas that have already been killed by this pipeline.

### Step 2: Store Ideas

Parse the ideation output and extract each idea. For each idea, write a self-contained markdown file to `.ideas/`:

- `.ideas/idea-1.md`
- `.ideas/idea-2.md`
- `.ideas/idea-3.md`

Create the `.ideas/` directory if it doesn't exist. **Overwrite** any existing files from a previous run.

Each file should contain the full idea description in structured format:

```markdown
# [Idea Name]

**One-liner:** [What it does in one sentence]
**Problem:** [The specific pain point]
**Solution:** [How this product solves it]
**Target user:** [Who pays for this, be specific]
**Revenue model:** [How it makes money]
**Distribution channel:** [Specific channel where customers already are]
**Estimated user base:** [Realistic number of addressable users]
**Key differentiator:** [Why this beats incumbents]
```

### Step 3: Present Ideas and Prompt

Display all 3 ideas to the user in a clear, numbered format. For each idea show:
- **Number** (1, 2, 3)
- **Name**
- **One-liner**
- **Target user**
- **Revenue model** (one line)

Then call `AskUserQuestion` with exactly 3 options:
- question: "Ideas stored in `.ideas/`. What would you like to do?"
- header: "3 ideas generated"
- Options:
  - label: "Evaluate one", description: "Pick an idea to run through /evaluate-idea"
  - label: "Evaluate all 3", description: "Run /evaluate-idea on each idea sequentially"
  - label: "Done", description: "I'll review the ideas first — run /evaluate-idea 1, 2, or 3 when ready"

**When "Evaluate one":** call `AskUserQuestion` again with one option per idea:
  - label: "1. [Idea 1 Name]", description: "[one-liner]"
  - label: "2. [Idea 2 Name]", description: "[one-liner]"
  - label: "3. [Idea 3 Name]", description: "[one-liner]"

  Then invoke Skill `evaluate-idea` with the selected number as args.

**When "Evaluate all 3":** invoke Skill `evaluate-idea` with `args: "1"`. After it completes, invoke again with `args: "2"`, then `args: "3"`. Between each, display: `"--- Evaluating idea [N] of 3 ---"`.

**When "Done":** end with: "Run `/evaluate-idea 1`, `/evaluate-idea 2`, or `/evaluate-idea 3` when ready."

## Important

- Do NOT invoke analyst/contrarian/advocate subagents — that's evaluate-idea's job.
- Do NOT run any evaluation logic — just generate, store, and present.
- Always overwrite `.ideas/idea-{1,2,3}.md` on each run — these are the "current batch."
- The ideation agent handles web search and rejected-ideas.md filtering internally.
