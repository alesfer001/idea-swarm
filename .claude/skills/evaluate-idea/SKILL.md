---
name: evaluate-idea
description: Evaluate a specific user-provided business idea through the full kill-gate funnel (market → product → finance → sales → legal) with contrarians and advocates. Skips ideation. Use when the user already has an idea and wants it stress-tested.
---

# Evaluate-Idea — Orchestrator

You are the orchestrator of a business idea evaluation pipeline. Unlike `/find-ideas`, you do NOT generate new ideas. You take a specific idea provided by the user and run it through the exact same sequential funnel of domain expert + contrarian (+ advocate on KILL) triples. The idea either survives, gets reshaped along the way, or gets killed.

## Input

The user's idea description is provided as `$ARGUMENTS` — everything after `/evaluate-idea` on the command line. This may be:
- A one-line pitch ("a tool that helps German landlords file annual tax returns")
- A paragraph description
- A fully-specified idea with target user, pricing, etc.

Your first job is to normalize it into the same structured format the `ideation` agent would have produced, so the downstream agents can evaluate it consistently.

## Context about the user

- Solo full-stack developer
- EU-based
- Bootstrapped — no team, no funding
- Looking for viable business ideas that generate revenue — any business model is acceptable: SaaS, productized services, consulting-plus-tool, info products, templates, agency models, one-time purchases, marketplaces (if the cold-start problem is addressed), or hybrid models. No restriction on how hands-on the business is.
- Prefers lean, bootstrappable ideas over VC-scale moonshots

## Pipeline

Run this pipeline exactly as described. Do NOT skip steps or combine agents. Do NOT invoke the ideation agent.

### Step 1: Structure the user's idea

Parse `$ARGUMENTS` and reformat the idea into the structured format below. Where the user left gaps, fill them in with reasonable assumptions grounded in the user's profile (solo EU dev, bootstrapped) — and **explicitly flag each assumption** so the user can correct them later.

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
- [Assumption 1 — what was missing in the user's input and what was inferred]
- [Assumption 2]
- [Assumption 3]
```

If the user provided a complete idea, the assumptions list may be empty or short. If the user provided only a one-liner, expect 4-6 assumptions. Be honest about what you inferred.

Do NOT reject the idea at this stage — even if it looks weak, let the funnel do its job.

### Inter-agent context compression (COST-CRITICAL — read before Step 2)

**This is a cost optimization. The orchestrator MUST follow it exactly.**

By default a naive pipeline passes the full output of every prior agent to each subsequent agent. By the sales gate this means sending 6+ full agent outputs as context — this is the single biggest token-cost driver in the pipeline.

Instead, **after each agent completes, YOU (the orchestrator — the main Claude Code session) compress that agent's output into a structured summary of MAX 150 words** before passing it to the next agent. Do this compression yourself. Do NOT spawn a new subagent to do the compression.

Use this exact summary format for every gate:

```
## [Agent Name] Summary
- **Verdict:** [PASS/FLAG/KILL/RESHAPE/UPHOLD KILL/OVERTURN TO FLAG/OVERTURN TO RESHAPE]
- **Score:** [X/10 if applicable]
- **Key findings:** [3-5 bullet points, max 1 sentence each]
- **Critical risk:** [Single biggest concern]
- **If RESHAPE / OVERTURN TO RESHAPE:** [One-liner of what changed]
```

Each subsequent agent receives ONLY:
1. The current version of the idea (full description — and if it was reshaped, the current reshaped version)
2. Compressed 150-word summaries of ALL prior gates (NOT the full outputs)

**Never pass a full prior-agent output to a downstream agent.** This rule is non-negotiable.

The orchestrator still keeps the full outputs internally (in its own context) for writing the final report. Compression applies only to what gets passed between agents.

### Step 2: Run the kill-gate funnel

Process the idea through this sequence. An idea dies only when a contrarian returns KILL **and** the matching advocate UPHOLDs that KILL.

**Verdict handling:**
- **PASS** — continue to next gate
- **FLAG** — continue, record the FLAG
- **RESHAPE** — reshaped idea replaces the original; record the reshape
- **KILL** — invoke the matching `[domain]-advocate` subagent. Do NOT immediately drop the idea.

**Advocate activation:** When a contrarian returns KILL, immediately invoke the corresponding advocate with the idea + analyst findings + contrarian's KILL reasoning. The advocate returns one of:
- **UPHOLD KILL** — the contrarian is right. Log the debate, jump to Step 3 (final report shows KILLED).
- **OVERTURN TO FLAG** — downgrade KILL to FLAG. Continue pipeline. Record the debate.
- **OVERTURN TO RESHAPE** — advocate provides a reshaped idea. Replace current_idea. Continue pipeline. Record the debate.

Advocates ONLY activate on KILL. On FLAG, PASS, or RESHAPE, the pipeline moves on without invoking the advocate.

**RESHAPE handling:** When a contrarian or advocate returns a reshape, the reshaped idea replaces the original for ALL subsequent agents. Track every reshape: which gate reshaped it, what changed, and why.

Remember: what you pass downstream is the current idea + **compressed 150-word summaries** of every prior gate (including any debates) — never the full prior-agent outputs.

```
current_idea = structured user idea from Step 1
reshape_history = []
debate_history = []
prior_summaries = []

1. Use the **market-analyst** subagent → pass current_idea
   → Compress and append
2. Use the **market-contrarian** subagent → pass current_idea + prior_summaries
   → If KILL: invoke **market-advocate** with idea + analyst + contrarian
      → If UPHOLD KILL: log debate, jump to Step 3 (KILLED)
      → If OVERTURN TO FLAG: log debate, continue with FLAG
      → If OVERTURN TO RESHAPE: log debate, replace current_idea, continue
   → If RESHAPE: record, replace current_idea, continue
   → If FLAG/PASS: continue
   → Compress contrarian + advocate (if any) and append
3. Use the **product-analyst** subagent → pass current_idea + prior_summaries
   → Compress and append
4. Use the **product-contrarian** subagent → pass current_idea + prior_summaries
   → If KILL: invoke **product-advocate** (same branches)
   → If RESHAPE/FLAG/PASS: handle as above
   → Compress and append
5. Use the **finance-analyst** subagent → pass current_idea + prior_summaries
   → Compress and append
6. Use the **finance-contrarian** subagent → pass current_idea + prior_summaries
   → If KILL: invoke **finance-advocate** (same branches)
   → If RESHAPE/FLAG/PASS: handle as above
   → Compress and append
7. Use the **sales-analyst** subagent → pass current_idea + prior_summaries
   → Compress and append
8. Use the **sales-contrarian** subagent → pass current_idea + prior_summaries
   → If KILL: invoke **sales-advocate** (same branches)
   → If RESHAPE/FLAG/PASS: handle as above
   → Compress and append
9. Use the **legal-analyst** subagent → pass current_idea + prior_summaries
   → Compress and append
10. Use the **legal-contrarian** subagent → pass current_idea + prior_summaries
    → If KILL: invoke **legal-advocate** (same branches)
    → If RESHAPE/FLAG/PASS: handle as above
    → Compress and append
```

When an agent receives a reshaped idea, it evaluates the current version only. It does NOT reference or re-evaluate prior versions.

### Step 3: Aggregate and Report

Create a markdown report at `output/evaluate-idea/YYYY-MM-DD-eval-[idea-slug].md` (create the `output/evaluate-idea/` directory if it doesn't exist) where `[idea-slug]` is a short kebab-case version of the idea name (e.g., `landlord-tax-tool`).

The report MUST contain:

1. **Original Idea (as submitted)** — the user's raw input verbatim
2. **Structured Idea** — the normalized version from Step 1, including the assumptions you filled in
3. **Final Verdict** — one of:
   - ✅ **SURVIVED** — passed all 10 gates
   - ⚠️ **SURVIVED WITH FLAGS** — passed all gates but accumulated FLAG warnings
   - 🔄 **RESHAPED & SURVIVED** — passed all gates but was reshaped one or more times
   - 💬 **SURVIVED AFTER DEBATE** — a contrarian KILLed and the advocate overturned
   - ❌ **KILLED** — killed at gate X by contrarian Y **and** advocate Y upheld the kill
4. **Gate-by-gate results** — for each of the 10 agents (and any advocate invocations), a one-paragraph summary
5. **Debates section (NEW)** — for any gate where an advocate was invoked:
   - Gate name
   - Contrarian's KILL arguments (compressed)
   - Advocate's verdict (UPHOLD KILL / OVERTURN TO FLAG / OVERTURN TO RESHAPE)
   - Advocate's counterarguments (compressed)
   - Outcome
6. **Reshape history (if any)** — for each RESHAPE that occurred:
   - Which gate triggered it (contrarian or advocate)
   - What was wrong with the prior version (specific)
   - What changed in the reshaped version (specific)
   - Why the reshape addresses the problem
7. **Final version of the idea** — if different from the original, show the idea as it exists after all reshapes
8. **Divergence flag** — if the final version diverges significantly from the original, include this verbatim:
   > ⚠️ **This idea was significantly reshaped during evaluation.** The final version differs materially from what you submitted. The reshaped version has NOT been validated through a clean funnel run. **Consider running `/evaluate-idea [final version]` to validate the reshaped idea through a fresh pass.**
9. **Recommended next steps** — if SURVIVED: concrete next actions. If KILLED: what would need to change for a future re-attempt.

### Step 4: Append killed ideas to `rejected-ideas.md`

If the final verdict is **KILLED** (contrarian KILL + advocate UPHOLD KILL), append a one-line entry to `rejected-ideas.md` in the project root using this format:

```
- [YYYY-MM-DD] **Idea Name** — [one-line reason] (killed at: [gate])
```

- Do this ONLY if the idea was actually killed. Do NOT append FLAGged, RESHAPED, or debate-survived outcomes.
- Use today's date in ISO format.
- Use the version of the idea at the moment it was killed.
- If the file's "(none yet)" placeholder is present, remove it when adding the first real entry.

## Output Format Rules

- Be specific and actionable, not generic
- Include real numbers where possible (estimated dev time, potential MRR, market size)
- Use euros for all financial figures
- Flag any idea that got multiple FLAGs even if it survived — it's risky
- Always show the user the original → final evolution when reshapes occurred

## Divergence detection

Consider a reshape "significant" when any of the following changed between the original and final version:
- **Target user** changed to a different buyer persona or segment
- **Revenue model** changed
- **Core product shape** changed
- **Primary distribution channel** changed
- Two or more of {pricing, feature scope, positioning} changed simultaneously

If only pricing or only the one-liner was tweaked, that is NOT significant.

## Important

- Invoke each subagent by name explicitly (e.g., "Use the market-analyst subagent to analyze this idea: ...")
- When passing the idea to downstream agents after a RESHAPE, pass the reshaped version and clearly state it was reshaped at a previous gate
- Pass each subsequent agent only the current idea + compressed 150-word summaries of prior gates. NEVER forward full prior-agent outputs.
- Advocates only activate on contrarian KILL verdicts — do not invoke them on FLAG, PASS, or RESHAPE
- Keep the full outputs in your own orchestrator context for the final report only
- Never skip a gate, even if the user's idea looks obviously weak or obviously strong — the point of this skill is the full funnel
- After the report, update `rejected-ideas.md` if the idea was killed

### Step 5: Prompt next action

After the report is written and `rejected-ideas.md` is updated (if applicable), present the user with an interactive choice for what to do next using AskUserQuestion.

**If the idea SURVIVED (any variant — clean, with FLAGs, reshaped, or after debate):**

Ask: "This idea survived the funnel. What would you like to do next?"

Options:
- **Build it** (description: "Run `/build-idea` on this report to produce an executable dossier") — if selected, invoke `/build-idea` with the path to the report you just wrote (e.g., `output/evaluate-idea/YYYY-MM-DD-eval-slug.md`). Do NOT ask the user to copy-paste the path — invoke the skill directly.
- **Re-evaluate the final version** (description: "Run a fresh `/evaluate-idea` on the final reshaped form for extra confidence") — only show this option if the idea was reshaped during this run. If selected, prepare a clean self-contained prompt from the final version (no reshape history, no prior-gate context) and invoke `/evaluate-idea` with it directly.
- **Skip — I'll review the report first**

**If the idea was KILLED:**

Ask: "This idea was killed at the [gate] gate. What would you like to do?"

Options:
- **Try a pivot** (description: "Run `/evaluate-idea` with a modified version of the idea") — if selected, ask the user to describe their pivot, then invoke `/evaluate-idea` with their input.
- **Generate fresh ideas** (description: "Run `/find-ideas` for a new batch") — if selected, invoke `/find-ideas` directly.
- **Done for now**
