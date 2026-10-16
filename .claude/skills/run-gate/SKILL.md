---
name: run-gate
description: Run a single evaluation gate (analyst + contrarian + advocate if KILL) on an idea. Primary gate executor for the evaluation pipeline. Reads state from .eval/, runs the gate, writes results, and suggests next steps.
---

# Run Gate — Primary Gate Executor

You run a single evaluation gate (analyst + contrarian, plus advocate if KILL) on an idea. All state lives on disk in `.eval/` — you read it, run the gate, write results, and hand back to the user.

## Input

`$ARGUMENTS` can be:

### Pipeline mode (`.eval/` exists)

- **No arguments or just a gate name** — run the gate indicated by `.eval/state.md` (or the specified gate)
- **`--auto`** — run ALL remaining gates sequentially without stopping. After each gate, check for KILL/RESHAPE before continuing to the next.

### Standalone mode (no `.eval/`)

- **`<gate> <idea-reference>`** — run a single gate on an idea without pipeline state. Useful for debugging.
  - gate: one of `market`, `product`, `finance`, `sales`, `legal`
  - idea-reference: a number (`1`, `2`, `3`) for `.ideas/idea-{N}.md`, a file path, or raw text

Examples:
- `/run-gate` — run next gate from .eval/state.md
- `/run-gate market` — run market gate (must match current gate in state.md, or standalone)
- `/run-gate --auto` — run all remaining gates automatically
- `/run-gate legal "a SaaS tool for German landlords"` — standalone, no .eval/

## Context about the user

- Solo full-stack developer, EU-based, bootstrapped
- Looking for viable, lean, bootstrappable business ideas

## Pipeline

### Step 1: Determine mode and parse input

**Check if `.eval/state.md` exists:**

**If yes (pipeline mode):**
1. Read `.eval/state.md` to get current gate, step, reshape depth, reshape budget, gates order
2. If state shows `status: completed` or `status: killed`, tell user the evaluation is already done and suggest generating the report
3. If `$ARGUMENTS` is `--auto`, set auto_mode = true
4. If `$ARGUMENTS` is a gate name, verify it matches `current_gate` in state.md. If not, warn the user and ask if they want to skip ahead or run the suggested gate.
5. If `$ARGUMENTS` is empty, use `current_gate` from state.md
6. Read `.eval/idea.md` for the current idea

**If no (standalone mode):**
1. Parse `$ARGUMENTS` as `<gate> <idea-reference>`
2. If malformed, use `AskUserQuestion` to ask for gate name and idea
3. Resolve the idea reference (number → `.ideas/idea-{N}.md`, path → read file, text → use as-is)
4. Normalize the idea into the structured format

### Step 2: Build prior context

**Pipeline mode:** Read the `## Summary` section from each completed gate file in `.eval/gates/` (NOT full output — summaries only). These become `prior_summaries`.

**Standalone mode:** No prior context.

**Founder channel fit (sales gate only, both modes):** if `learnings.md` exists, append its `## What this means for new ideas` section to the sales analyst and sales contrarian prompts under a `## Founder channel fit` heading, with this instruction:
- Analyst: recommend only channels it allows; say which allowed channel reaches the first customers.
- Contrarian: if the idea's first customers can only be reached through a channel it rules out (e.g. cold calls, in-person sales), that is a structural distribution problem: return RESHAPE with an allowed channel if one exists, otherwise KILL.

### Step 3: Run the gate

Read `pipeline.md` to confirm gate order and get the gate numbering.

#### 3a: Run the analyst

1. Read `.eval/idea.md` (or the standalone idea)
2. Use the **[gate]-analyst** subagent → pass current idea + prior_summaries
3. **Compress** the analyst output into a 150-word summary (you do this, not a subagent)
4. **Pipeline mode:** Write to `.eval/gates/NN-[gate]-analyst.md` using the gate file format. **The `## Full Output` section MUST contain the complete agent output verbatim — no placeholders.**
5. **Pipeline mode:** Update `.eval/state.md`: set `current_step: contrarian`

#### 3b: Run the contrarian

1. Read prior summaries from `.eval/gates/` (including the analyst just written)
2. Read `.eval/idea.md` for the current idea
3. If `reshape_depth >= reshape_budget` (from state.md), append to the agent prompt: `**RESHAPE BUDGET EXHAUSTED.** You MUST NOT return RESHAPE. Use FLAG with specific recommendations instead.`
4. If `reshape_depth >= 1`, prepend to the idea: `**Reshape context:** This idea was reshaped [N] time(s). Evaluate AS PRESENTED.`
5. Use the **[gate]-contrarian** subagent → pass current idea + prior summaries
6. **Pipeline mode:** Write to `.eval/gates/NN-[gate]-contrarian.md`. **The `## Full Output` section MUST contain the complete agent output verbatim — no placeholders.**

**Handle the verdict:**

- **PASS / FLAG:** Update state.md, mark gate as complete, advance `current_gate` to next gate. If this was the last gate, go to Step 4 (completion).
- **KILL:** Go to Step 3c (advocate).
- **RESHAPE:**
  - If `reshape_depth < reshape_budget`: go to Step 3d (reshape).
  - If `reshape_depth >= reshape_budget`: convert to FLAG, log "Reshape budget exhausted — contrarian's reshape recorded as FLAG." Update state, advance to next gate.

#### 3c: Run the advocate (only on KILL)

1. Invoke the **[gate]-advocate** with the idea + analyst summary + contrarian's KILL reasoning
2. **Pipeline mode:** Write to `.eval/gates/NNb-[gate]-advocate.md`. **Full Output must be complete and verbatim.**

**Handle the advocate verdict:**
- **UPHOLD KILL:** Update state.md: `status: killed`, `killed_at: [gate]`. Go to Step 5 (killed).
- **OVERTURN TO FLAG:** Log the debate in state.md `## Debates` section. Mark gate as FLAG. Advance to next gate.
- **OVERTURN TO RESHAPE:**
  - If `reshape_depth < reshape_budget`: go to Step 3d (reshape).
  - If `reshape_depth >= reshape_budget`: convert to FLAG. Log "Reshape budget exhausted — advocate's reshape recorded as FLAG." Advance to next gate.

#### 3d: Reshape

1. Write the reshape record to `.eval/reshapes/reshape-[N].md` (what changed, which gate, why)
2. Update `.eval/idea.md` with the reshaped idea
3. Increment `reshape_depth` in state.md
4. Increment `pass_number` in state.md
5. Reset all gate statuses to `pending` in state.md
6. Delete all files in `.eval/gates/` (clean pass)
7. Set `current_gate` to the first gate in the pipeline
8. Set `current_step` to `analyst`

**If auto_mode:** continue running from the top (restart the gate loop from the first gate).
**If manual mode:** tell the user the idea was reshaped and suggest `/run-gate market` to restart.

### Step 4: Gate or pipeline complete

**If this was the last gate (all gates done):**

Update state.md: `status: completed`.

Tell the user all gates are done and suggest:

```
All gates complete. Run /finalize-evaluation to generate the final report.
You can /clear first to save context.
```

**If the idea was killed:**

Update state.md: `status: killed`, `killed_at: [gate]`.

Tell the user and suggest:

```
This idea was KILLED at the [gate] gate. Run /finalize-evaluation to generate the report.
```

**If more gates remain:**

Go to Step 5 (prompt next action).

### Step 5: Prompt next action (gate done, more gates remain)

**If auto_mode and more gates remain:** proceed directly to the next gate (loop back to Step 3 with the next gate).

**If auto_mode and all gates done (or killed):** invoke Skill `finalize-evaluation` to generate the final report.

**If manual mode and the idea was killed:**

Display verdict summary and suggest `/finalize-evaluation` to generate the report.

**If manual mode and the idea was reshaped:**

Display what changed and tell the user:

```
Idea reshaped at [gate] gate (reshape [N]/[budget]).
The funnel restarts from gate 1 with the new version.

Next: /run-gate market    (or /clear first to save context)
```

**If manual mode, gate passed (PASS/FLAG), more gates remain:**

Display a concise summary:
- Gate name and verdict
- Analyst key findings (3 bullets)
- Contrarian verdict + key concerns (3 bullets)
- Advocate outcome (if invoked)

Then determine the next gate from `gates_order` in state.md and tell the user:

```
[Gate] gate complete ([verdict]). Next gate: [next-gate]

  /run-gate [next-gate]       ← run next gate
  /run-gate --auto            ← run all remaining gates
  
You can /clear first to save context.
```

## Gate file format

Every gate result file in `.eval/gates/` MUST use this format:

```markdown
---
gate: [domain]-[analyst|contrarian|advocate]
verdict: [PASS|FLAG|KILL|RESHAPE|UPHOLD KILL|OVERTURN TO FLAG|OVERTURN TO RESHAPE]
score: [X/10 or n/a]
pass_number: [N]
---

## Summary
- **Verdict:** [...]
- **Score:** [X/10]
- **Key findings:** [3-5 bullet points, max 1 sentence each]
- **Critical risk:** [Single biggest concern]
- **If RESHAPE / OVERTURN TO RESHAPE:** [One-liner of what changed]

## Full Output
[MANDATORY: Paste the COMPLETE agent output here — every word, every section, every bullet.
Do NOT write placeholders like "(see above)" or "(see agent output)".
Do NOT summarize or truncate. This is the ONLY permanent record of what the agent said.]
```

## Gate numbering

Gates are numbered sequentially based on their position in the pipeline config:
- Gate 1 (market): `01-market-analyst.md`, `02-market-contrarian.md`, `02b-market-advocate.md`
- Gate 2 (product): `03-product-analyst.md`, `04-product-contrarian.md`, `04b-product-advocate.md`
- Gate 3 (finance): `05-finance-analyst.md`, `06-finance-contrarian.md`, `06b-finance-advocate.md`
- Gate 4 (sales): `07-sales-analyst.md`, `08-sales-contrarian.md`, `08b-sales-advocate.md`
- Gate 5 (legal): `09-legal-analyst.md`, `10-legal-contrarian.md`, `10b-legal-advocate.md`

## Important

- **All state on disk.** Read from `.eval/` files, don't rely on your context for prior gate results.
- **Build prior summaries from disk.** For each gate, read `## Summary` from completed gate files — never pass full outputs to agents.
- Invoke each subagent by name explicitly
- Advocates only activate on contrarian KILL verdicts
- Update `.eval/state.md` after every step (analyst, contrarian, advocate) so the pipeline is resumable
- In standalone mode (no `.eval/`): do NOT write to `output/` or `rejected-ideas.md`, just display results
- In pipeline mode: always write gate files and update state
