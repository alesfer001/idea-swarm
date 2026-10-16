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

## Input

`$ARGUMENTS` is optional:
- **A preset name** (e.g. `/find-ideas digital-products`): if it matches a `### <name>` heading under `## Presets` in `pipeline.md`, use that preset's **Brief** as the focus and its **Searches** as the search budget.
- **Free text** (e.g. `/find-ideas "self-serve digital products"`): a one-line focus.
- **Empty:** an open run.

## Pipeline

### Step 1: Generate Ideas

Use the **ideation** subagent to generate exactly 3 business ideas. The ideation agent follows a demand-first, search-first process and will read `rejected-ideas.md` before generating, to avoid proposing ideas that have already been killed by this pipeline.

If a preset was given, start the subagent prompt with its Brief verbatim and add `For this run you may use up to <Searches> WebSearch queries.` If free-text focus was given, start with `Focus: <focus>`. Otherwise send no Focus line.

The ideation agent has no file access, so build its prompt from disk:
- **Founder channel fit:** if `learnings.md` exists, include its `## What this means for new ideas` section verbatim under a `## Founder channel fit` heading.
- **Already tried:** include the `## Rejected Ideas` entries from `rejected-ideas.md`, plus the idea names and one-liners from every report in `output/evaluate-idea/` and from the `## Log` table in `learnings.md`, as a do-not-repeat list.

### Step 2: Store Ideas

If `.ideas/idea-*.md` files exist from a previous run, move them to `.ideas/previous/` first (replacing whatever is there), so the last batch stays available as `.ideas/previous/idea-N.md`.

Parse the ideation output and extract each idea. For each idea, write a self-contained markdown file to `.ideas/`:

- `.ideas/idea-1.md`
- `.ideas/idea-2.md`
- `.ideas/idea-3.md`

Create the `.ideas/` directory if it doesn't exist.

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
**Focus fit:** [Only when a focus was given]
**Channel fit:** [How the first customers are reached within learnings.md, when present]
**Cheapest marketing test:** [The ideation agent's proposed test and signal]
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
- `.ideas/idea-{1,2,3}.md` is the current batch; the one before it lives in `.ideas/previous/`.
- The ideation agent handles web search; you pass it the rejected/tried ideas and founder channel fit (Step 1), since it can't read files.
