---
name: finalize-evaluation
description: Generate the final evaluation report from completed gate results in .eval/. Reads all gate files, writes the report to output/evaluate-idea/, updates rejected-ideas.md if killed, and cleans up .eval/.
---

# Finalize Evaluation — Report Generator

You generate the final evaluation report from completed gate results. All gate data lives on disk in `.eval/` — you read it, compile the report, and clean up.

## Input

No arguments needed. Everything is read from `.eval/`.

## Prerequisites

Check `.eval/state.md` exists. If not, tell the user there's no evaluation to finalize and suggest `/evaluate-idea`.

Check the `status` field in `.eval/state.md`:
- `completed` or `killed` → proceed
- `in_progress` → tell the user the evaluation isn't done yet, show which gate is next, and suggest `/run-gate`

## Pipeline

### Step 1: Read all state from disk

Read these files:
- `.eval/state.md` — status, reshape depth, debates
- `.eval/idea.md` — final version of the idea
- `.eval/original-idea.md` — original idea as submitted
- All files in `.eval/gates/` — read both `## Summary` and `## Full Output` sections from each
- All files in `.eval/reshapes/` — reshape records (if any)

### Step 2: Determine the final verdict

Based on state.md and gate results:

- **SURVIVED** — all gates passed with PASS, no FLAGs, no debates, no reshapes
- **SURVIVED WITH FLAGS** — all gates passed but one or more returned FLAG
- **RESHAPED & SURVIVED** — reshaped one or more times, then passed all gates
- **SURVIVED AFTER DEBATE** — at least one contrarian returned KILL but the advocate overturned it
- **KILLED** — killed at a gate (contrarian KILL + advocate UPHOLD KILL)

Combine as needed (e.g., reshaped AND survived after debate AND with flags).

### Step 3: Write the final report

Create a markdown report at `output/evaluate-idea/YYYY-MM-DD-eval-[idea-slug].md` (create directory if needed).

The report MUST contain:

1. **Original Idea (as submitted)** — from `.eval/original-idea.md`
2. **Structured Idea** — the normalized version with assumptions
3. **Final Verdict** — using the verdict determined in Step 2
4. **Gate-by-gate results** — for each gate file, read `## Full Output` and write a one-paragraph summary per agent (analyst, contrarian, advocate if present)
5. **Debates section** — for any gate where an advocate was invoked, summarize the analyst → contrarian → advocate exchange
6. **Reshape history** — from `.eval/reshapes/`, describe each reshape (what changed, why, which gate triggered it)
7. **Final version of the idea** — from `.eval/idea.md` (may differ from original if reshaped)
8. **Divergence flag** — if reshaped:
   - All reshapes got clean re-runs: note that each reshaped version was re-evaluated through a clean funnel pass from Gate 1
   - Reshape budget hit with forced FLAGs: note the budget was hit and how many reshape attempts were forced to FLAG
9. **Recommended next steps**

### Step 4: Update rejected-ideas.md (if killed)

If the idea was killed (contrarian KILL + advocate UPHOLD KILL), append to `rejected-ideas.md` in the project root:

```
- [YYYY-MM-DD] **Idea Name** — [reason] (killed at: [gate])
```

### Step 5: Clean up .eval/

**Delete the entire `.eval/` directory.** The report in `output/evaluate-idea/` is the permanent record — `.eval/` was working state only.

### Step 6: Prompt next action

**If SURVIVED (any variant):**

Display the verdict and report path, then use `AskUserQuestion`:
- "Build it" → suggest `/build-idea <report-path>`
- "Evaluate another idea" → suggest `/evaluate-idea`
- "Done" → stop

**If KILLED:**

Display the verdict and report path, then use `AskUserQuestion`:
- "Try a pivot" → ask for pivot description, suggest `/evaluate-idea <pivot>`
- "Generate fresh ideas" → suggest `/find-ideas`
- "Done" → stop

## Important

- This skill does NOT run any gates or invoke any analyst/contrarian/advocate agents
- All data comes from `.eval/` files on disk — do not rely on conversation context
- The `## Full Output` sections in gate files are the ONLY source of truth for what agents said
- Always clean up `.eval/` after writing the report
