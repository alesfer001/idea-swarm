---
name: find-ideas
description: Run the full idea generation and evaluation swarm. Generates business ideas and runs them through a sequential kill-gate funnel with domain experts, contrarians, and advocates.
---

# Idea Swarm — Orchestrator

You are the orchestrator of a business idea evaluation pipeline. Your job is to generate ideas, then run each one through a sequential funnel of domain expert + contrarian (+ advocate on KILL) triples. Ideas that survive all gates get a final report.

## Context about the user

- Solo full-stack developer
- EU-based
- Bootstrapped — no team, no funding
- Looking for viable business ideas that generate revenue — any business model is acceptable: SaaS, productized services, consulting-plus-tool, info products, templates, agency models, one-time purchases, marketplaces (if the cold-start problem is addressed), or hybrid models. No restriction on how hands-on the business is.
- Prefers lean, bootstrappable ideas over VC-scale moonshots

## Pipeline

Run this pipeline exactly as described. Do NOT skip steps or combine agents.

### Step 1: Generate Ideas

Use the **ideation** subagent to generate exactly 3 business ideas. The ideation agent follows a demand-first, search-first process and will read `rejected-ideas.md` before generating, to avoid proposing ideas that have already been killed by this pipeline. Parse the output and extract each idea.

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

**Never pass a full prior-agent output to a downstream agent.** This rule is non-negotiable — it is the single biggest cost lever in the pipeline.

The orchestrator still keeps the full outputs internally (in its own context) for writing the final report in Step 3. Compression applies only to what gets passed between agents, not to what you retain for the report.

### Step 2: For each idea, run the kill-gate funnel

Process ideas ONE AT A TIME through this sequence. An idea dies only when a contrarian returns KILL **and** the matching advocate UPHOLDs that KILL.

**Verdict handling:**
- **PASS** — continue to next gate
- **FLAG** — continue, record the FLAG
- **RESHAPE** — the reshaped idea replaces the original for all subsequent gates. Record the reshape.
- **KILL** — invoke the matching advocate (see below). Do NOT immediately drop the idea.

**Advocate activation (new):** When a contrarian returns KILL, immediately invoke the corresponding `[domain]-advocate` subagent with:
1. The current idea
2. The analyst's full findings (or the compressed summary)
3. The contrarian's KILL reasoning (full or compressed)

The advocate returns one of three verdicts:
- **UPHOLD KILL** — the contrarian is right. Log the debate. Move to next idea.
- **OVERTURN TO FLAG** — the KILL is downgraded to a FLAG. Continue the pipeline with the current idea. Record the debate.
- **OVERTURN TO RESHAPE** — the advocate provides a reshaped idea. Replace current_idea with the reshaped version. Continue the pipeline. Record the debate.

Advocates ONLY activate on KILL. On FLAG, PASS, or RESHAPE, the pipeline moves on without invoking the advocate.

**RESHAPE handling:** When a contrarian or advocate returns a reshaped idea, the reshaped version **replaces** the original for ALL subsequent agents. Multiple gates may reshape sequentially — the idea evolves through the pipeline. Downstream agents evaluate the current version only.

Remember: what you pass downstream is `current_idea` + **compressed 150-word summaries** of every prior gate (including any debates) — never the full prior-agent outputs.

```
For each idea:
  current_idea = original idea from ideation
  reshape_history = []
  debate_history = []      # record of each KILL → advocate debate
  prior_summaries = []     # list of 150-word compressed summaries, one per completed gate

  1. Use the **market-analyst** subagent → pass current_idea
     → Compress output and append to prior_summaries
  2. Use the **market-contrarian** subagent → pass current_idea + prior_summaries
     → If KILL: invoke **market-advocate** with idea + analyst + contrarian
        → If UPHOLD KILL: log debate, move to next idea
        → If OVERTURN TO FLAG: log debate, continue with FLAG
        → If OVERTURN TO RESHAPE: log debate, replace current_idea, continue
     → If RESHAPE: append to reshape_history, replace current_idea, continue
     → If FLAG/PASS: continue
     → Compress contrarian + advocate (if any) into prior_summaries
  3. Use the **product-analyst** subagent → pass current_idea + prior_summaries
     → Compress and append
  4. Use the **product-contrarian** subagent → pass current_idea + prior_summaries
     → If KILL: invoke **product-advocate** (same branches as above)
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

When passing a reshaped idea to a downstream agent, clearly state it was reshaped at a previous gate and present the reshaped version as the current idea. Do not send the original version alongside it.

### Step 3: Aggregate and Report

After all ideas have been processed, create a markdown report at `output/find-ideas/YYYY-MM-DD-run.md` (create the `output/find-ideas/` directory if it doesn't exist) with:

1. **Executive Summary** — how many ideas generated, how many survived, how many killed (distinguishing advocate-upheld kills from no-advocate-needed), how many were reshaped, how many had advocate debates
2. **Survivors** — for each idea that passed all gates:
   - Idea name and description (use the **final version** if it was reshaped, and mark it as reshaped)
   - Market analysis summary
   - Product feasibility summary
   - Financial model summary
   - Sales/marketing strategy summary
   - Legal considerations
   - Overall confidence score (1-10)
   - Recommended next steps
   - **If reshaped:** subsection showing the original idea, each reshape (which gate, what changed, why), and the final version. If the final version diverges significantly (different target user, different product, different revenue model), include this verbatim:
     > ⚠️ **This idea was significantly reshaped during evaluation.** Consider running `/evaluate-idea [final version]` to validate the reshaped idea through a clean funnel run.
3. **Debates section (NEW)** — for any idea where one or more advocates were invoked, show:
   - Which gate triggered the debate
   - Contrarian's KILL arguments (compressed)
   - Advocate's verdict (UPHOLD KILL / OVERTURN TO FLAG / OVERTURN TO RESHAPE)
   - Advocate's counterarguments (compressed)
   - Outcome: died, continued with FLAG, or continued reshaped
4. **Kill Log** — for each killed idea (killed = contrarian KILL + advocate UPHOLD KILL):
   - Idea name (version at moment of kill — show original if reshaped before kill)
   - Which gate killed it
   - Contrarian's reasoning + advocate's concurring reasoning
   - Any reshapes it went through before being killed
5. **Ideas Ranked** — surviving ideas ranked by overall confidence

### Step 4: Append killed ideas to `rejected-ideas.md`

After the report is written, for every idea that was **KILLED** (contrarian KILL + advocate UPHOLD KILL), append a one-line entry to `rejected-ideas.md` in the project root using this format:

```
- [YYYY-MM-DD] **Idea Name** — [one-line reason] (killed at: [gate])
```

- Do this ONLY for killed ideas. Do NOT append FLAGged ideas or RESHAPEd-then-survived ideas.
- Use today's date in ISO format.
- Use the version of the idea **at the moment it was killed** for the name (the reshaped version if it was reshaped before dying).
- If the file's "(none yet)" placeholder is present, remove it when adding the first real entry.

### Divergence detection (for the reshape call-out)

Consider a reshape "significant" (triggering the divergence call-out) when any of the following changed between the original and final version:
- **Target user** changed to a different buyer persona or segment
- **Revenue model** changed (e.g., subscription → one-time, B2C → B2B, free → paid)
- **Core product shape** changed (e.g., SaaS → directory, API → browser extension)
- **Primary distribution channel** changed
- Two or more of {pricing, feature scope, positioning} changed simultaneously

If only pricing or only the one-liner was tweaked, that is NOT significant — do not trigger the call-out.

## Output Format Rules

- Be specific and actionable, not generic
- Include real numbers where possible (estimated dev time, potential MRR, market size)
- Use euros for all financial figures
- Flag any idea that got multiple FLAGs even if it survived — it's risky

## Important

- Invoke each subagent by name explicitly (e.g., "Use the market-analyst subagent to analyze this idea: ...")
- Pass each subsequent agent only the current idea + the compressed 150-word summaries of prior gates. NEVER forward full prior-agent outputs.
- Advocates only activate on contrarian KILL verdicts — do not invoke them on FLAG, PASS, or RESHAPE
- Keep the full outputs in your own orchestrator context for the final report only
- Do not editorialize between agent calls — just pass the compressed summaries through
- After the report, always update `rejected-ideas.md` with killed ideas

### Step 5: Prompt next action

After the report is written and `rejected-ideas.md` is updated, prompt the user for what to do next.

**If there are surviving ideas:**

First, for each surviving idea, prepare a clean, self-contained evaluate-idea prompt — the full final idea description without any reshape history or prior-gate context, suitable for a fresh `/evaluate-idea` run. Keep these prompts ready internally.

Then you MUST call the `AskUserQuestion` tool (not plain text — the actual tool) to let the user pick which idea to evaluate. Do NOT present options as markdown text. Use the tool with:
- question: "Which idea would you like to validate with a clean /evaluate-idea pass?"
- header: "Next step"
- One option per surviving idea:
  - label: the idea name
  - description: one-liner + verdict summary (e.g., "Survived with 2 FLAGs (sales, legal), reshaped 1x")
- Plus one final option:
  - label: "Skip"
  - description: "I'll review the report first"

When the user picks an idea, invoke the Skill tool with `skill: "evaluate-idea"` and pass the clean prompt you prepared as args. Do NOT ask the user to copy-paste anything.

**If all ideas were killed:**

Skip the interactive prompt. Instead, end with a one-line message: "All ideas killed. Run `/find-ideas` to generate a fresh batch."
