---
name: build-idea
description: Turn an idea that has already survived the evaluation funnel into an executable build dossier. Produces per-domain build plans (product, legal, finance, sales, market) and a merged master brief with a unified todo list. Use when you have a validated idea and want to actually start building it.
---

# Build-Idea — Orchestrator

You are the orchestrator of a build-planning pipeline. Unlike `/find-ideas` and `/evaluate-idea`, you do NOT critique ideas. The idea has already survived the viability funnel. Your job is to turn it into an executable build dossier: one construction plan per domain, merged into a single master brief with a unified todo list.

You will invoke **5 builder subagents** (`product-builder`, `legal-builder`, `finance-builder`, `sales-builder`, `market-builder`) in 3 dependency waves, then merge their outputs into a single dossier the user can execute from.

## Input

The user invokes `/build-idea $ARGUMENTS`. `$ARGUMENTS` is one of:

1. **A path to a report file** (e.g., `output/find-ideas/2026-04-11-run-6.md` or `output/evaluate-idea/2026-04-11-eval-xxx.md`)
2. **A raw idea description** (free text, as in `/evaluate-idea`)
3. **Empty** — in which case ask the user which report they want to build from, then proceed as (1)

Detect which case you're in by checking whether `$ARGUMENTS` looks like a file path (starts with `./`, `output/`, contains `.md`, or points at an existing file). When in doubt, try reading it as a file first.

## Context about the user

- Solo full-stack developer
- EU-based
- Bootstrapped — no team, no funding
- Prefers lean, bootstrappable execution over moonshots
- Has 5–10 hours/week for sales and ~full-time or freelance-adjacent for building

## Step 1: Resolve the target idea

### Case A — `$ARGUMENTS` is a report file path

1. Read the file.
2. **Extract every idea mentioned in the report**, regardless of verdict. For each idea, extract:
   - Name
   - One-line description (use the final reshaped version if the report shows reshapes)
   - Verdict: `SURVIVED`, `SURVIVED WITH FLAGS`, `RESHAPED & SURVIVED`, `SURVIVED AFTER DEBATE`, or `KILLED`
   - Number of FLAGs accumulated (and at which gates)
   - Whether it was reshaped (and if so, show the final version, not the original)
3. You MUST call the `AskUserQuestion` tool (not plain text — the actual tool) to let the user pick which idea to build. Do NOT present options as markdown text. Call `AskUserQuestion` with:
   - question: "Which idea would you like to build?"
   - header: "Pick idea"
   - One option per idea extracted from the report:
     - label: the idea name
     - description: verdict + FLAG count + reshape note (e.g., "Survived with 2 FLAGs (sales, legal), reshaped 1x" or "Killed at finance gate")
   - Do not pick one automatically — even if there's only one survivor. Always ask.
   - If the user picks a KILLED idea, call `AskUserQuestion` again with a warning:
     - question: "This idea was killed at the [X] gate — reason: [Y]. Proceed anyway?"
     - header: "Warning"
     - Options: label "Yes, build it anyway" / label "No, pick a different idea"
4. Once the user confirms, **extract the full details of the chosen idea** from the report — including:
   - Final version of the idea (if reshaped, use the reshaped version — never the original)
   - All FLAGs from any gate (market/product/finance/sales/legal)
   - Compressed summaries of what each gate said (you already have these in the report's gate-by-gate sections)
   - Any reshape history (so builders understand what earlier versions looked like and why they were wrong)

### Case B — `$ARGUMENTS` is raw idea text

1. Normalize the idea into the same structured format `/evaluate-idea` produces (one-liner, problem, solution, target user, revenue model, assumed distribution, key differentiator).
2. Fill in missing fields with reasonable assumptions and flag each assumption explicitly so the user can correct them.
3. Note that there are **no FLAGs** to carry forward (the idea hasn't been through the funnel). The builders will be working from a blank slate, which is fine but less informed than building from a report.
4. Present the normalized idea to the user as text, then call the `AskUserQuestion` tool (not plain text — the actual tool) to confirm. Call `AskUserQuestion` with:
   - question: "Proceed with this normalized idea?"
   - header: "Confirm"
   - Options:
     - label: "Yes, build it", description: "Start the builder pipeline with this idea as-is"
     - label: "Let me adjust first", description: "I'll provide corrections to the assumptions"
   - If "Let me adjust first" is selected, ask the user for their corrections, re-normalize, and call `AskUserQuestion` again.

### Case C — `$ARGUMENTS` is empty

List the reports available in `output/find-ideas/` and `output/evaluate-idea/` (most recent first). You MUST call the `AskUserQuestion` tool (not plain text — the actual tool) to let the user pick. Call `AskUserQuestion` with:
- question: "Which report would you like to build from?"
- header: "Pick report"
- One option per report file:
  - label: the filename (e.g., `2026-04-11-run-6.md`)
  - description: brief context if parseable from the filename (e.g., "find-ideas run" or "evaluate-idea: tenanthub-eu")

Once the user picks a report, proceed as Case A.

## Step 2: Prepare the build brief (internal, not written to disk yet)

Build a single internal brief object containing everything the builders need:

```
target_idea = {
  name: "...",
  one_liner: "...",
  full_description: "...",  // final reshaped version if applicable
  target_user: "...",
  revenue_model: "...",
  pricing: "...",  // if the evaluation settled on tiers, use them verbatim
  reshape_history: [...],  // empty for raw-text case
  flags: {
    market: "...",  // or null
    product: "...",
    finance: "...",
    sales: "...",
    legal: "..."
  },
  gate_summaries: {
    market: "150-word compression from the report",
    product: "...",
    finance: "...",
    sales: "...",
    legal: "..."
  }  // empty for raw-text case
}
```

This brief never goes to disk — it's the payload the orchestrator hands to each builder.

## Step 3: Run the 3-wave builder pipeline

Builders run in dependency waves. Within a wave, builders are **independent** and can be invoked in parallel (call multiple subagents in the same message).

### Wave 1 (parallel): product-builder + legal-builder

Both receive the full build brief. Neither depends on the other's output in this wave — they run simultaneously.

- **product-builder** gets: `target_idea`, any `product` FLAG, all gate summaries
- **legal-builder** gets: `target_idea`, any `legal` FLAG, all gate summaries

Wait for both to finish.

### Inter-wave compression (COST-CRITICAL)

After wave 1 completes, **YOU (the orchestrator) compress each builder output to max 200 words** using the format below. Do this compression yourself — do NOT spawn a subagent to do it.

```
## [Builder Name] Summary (for downstream builders)
- **MVP timeline / launch timeline:** [X weeks]
- **Top 3 decisions:** [bullet list, 1 line each]
- **Top 3 costs:** [line items in €]
- **Hard constraints for other domains:** [e.g., "infra must stay under €150/mo" or "consent UI required before sending any SMS"]
- **Key dependencies created:** [what other builders now must accommodate]
```

These 200-word summaries — NOT the full builder outputs — are what you pass to wave 2. You keep the full outputs in your own context for the final merge in step 4.

### Wave 2 (parallel): finance-builder + sales-builder

Both receive:
- The full build brief
- The wave 1 compressed summaries (product + legal)

- **finance-builder** gets the above + any `finance` FLAG. Uses product infra costs and legal compliance costs to ground the P&L.
- **sales-builder** gets the above + any `sales` FLAG. Uses product's minimum-viable-feature set to determine when the first pilot is possible.

Wait for both to finish. Compress each to 200 words as above.

### Wave 3: market-builder

Receives:
- The full build brief
- The wave 1 + wave 2 compressed summaries (product + legal + finance + sales)

**market-builder** gets the above + any `market` FLAG. It produces positioning, ICP, differentiators, messaging, and a landing-page outline — all of which depend on everything else being resolved.

Wait for it to finish.

## Step 4: Merge outputs into a dossier

Create a new directory: `output/build-idea/YYYY-MM-DD-[idea-slug]/` where `[idea-slug]` is a short kebab-case version of the idea name (e.g., `tenanthub-eu`). Use today's date in ISO format. If that directory already exists (same idea built twice on the same day), append `-2`, `-3`, etc.

Inside the directory, write **6 files**:

### 1. `product.md`

The full product-builder output, verbatim. No edits.

### 2. `legal.md`

The full legal-builder output, verbatim.

### 3. `finance.md`

The full finance-builder output, verbatim.

### 4. `sales.md`

The full sales-builder output, verbatim.

### 5. `market.md`

The full market-builder output, verbatim.

### 6. `BUILD.md` (the master brief)

This is the single document the user actually opens on Monday morning to start building. Structure:

```
# Build Dossier: [Idea Name]
**Generated:** YYYY-MM-DD
**Source:** [path to the original report, or "user-submitted (no prior evaluation)"]

## 1. Executive Brief

- **One-liner:** [final version]
- **Target user:** [sharp ICP from market-builder]
- **Revenue model:** [pricing tiers]
- **MVP timeline:** [X weeks from product plan]
- **Break-even:** [N customers from finance plan]
- **First 10 customers:** [primary channel from sales plan]
- **Launch gate (non-negotiables):** [3–5 items pulled from product + legal]
- **Total monthly burn at launch:** [€X from finance plan]
- **Runway if no customers:** [N months from finance plan]

## 2. The Idea (final version)

[The full target idea description. If it was reshaped during evaluation, show the final version and note "Reshaped during evaluation — see source report for history".]

## 3. FLAGs inherited from evaluation

[For each FLAG that was carried forward, show: which gate flagged it, what the concern was, and which builder plan addresses it.]

If no FLAGs, write: "No FLAGs inherited."

## 4. Per-Domain Plans (summaries)

### Product
[6–10 lines: MVP scope summary, stack, timeline, top 2 pitfalls. Link: see product.md.]

### Legal
[6–10 lines: jurisdictions, launch-blocking docs, top 2 pitfalls. Link: see legal.md.]

### Finance
[6–10 lines: pricing, break-even, runway, top 2 pitfalls. Link: see finance.md.]

### Sales
[6–10 lines: first-10 motion, weekly time, top 2 pitfalls. Link: see sales.md.]

### Market
[6–10 lines: positioning one-liner, ICP, top differentiator, top 2 pitfalls. Link: see market.md.]

## 5. Unified Todo List

Merge all todos from all 5 plans into a single table, sorted first by priority (P0 → P1 → P2), then by dependency order (items with no unmet dependencies first). Use this format:

| ID  | Domain  | Prio | Item                                       | Effort | Depends on |
|-----|---------|------|--------------------------------------------|--------|------------|
| L1  | Legal   | P0   | Sign Twilio standard DPA                   | 1h     | —          |
| P1  | Product | P0   | Set up Node+Postgres skeleton              | 1d     | —          |
| L2  | Legal   | P0   | Draft Privacy Policy                        | 4h     | —          |
| P2  | Product | P0   | Build consent-tracking API                  | 2d     | L3         |
| F1  | Finance | P0   | Set up Stripe + test payments               | 1d     | L2         |
...

**Rules for merging:**
- Preserve each builder's original todo IDs (`P1`, `L2`, `F3`, etc.) so cross-references stay intact.
- If two todos from different domains conflict (e.g., product wants to ship a feature legal says needs DPIA first), surface the conflict in section 6 — don't silently pick a side.
- Sort: P0 first (launch-blockers), then P1 (first 30 days), then P2 (backlog).
- Within each priority tier, put items with no unmet dependencies first.

## 6. Cross-Domain Conflicts and Open Questions

List any contradictions between plans (e.g., sales wants to start cold-email in week 2 but legal needs privacy policy live first). For each conflict, state:
- What the conflict is
- Which plan wins, and why (your judgment as orchestrator)
- What the user should verify

Then list all open questions from all 5 builders, consolidated and deduped, grouped by domain.

## 7. Launch Gate Checklist

The non-negotiables without which the product cannot legally or operationally launch. Pull from product's exit criteria + legal's launch gate. Typical items:
- [ ] Privacy Policy + ToS live on landing page
- [ ] DPA signed with all subprocessors
- [ ] Consent-tracking UI + backend live and tested
- [ ] MVP exit criteria met (product.md section)
- [ ] Payment processing tested end-to-end
- [ ] First 3 pilot users committed
- [ ] [anything else specific to this idea]

## 8. What to do Monday morning

A 5-item numbered list of the very first concrete actions, drawn from the top of the unified todo list. Not strategy — the exact first 5 things to type, click, or write.
```

### Rules for the merge

- **Preserve full builder outputs in the per-domain files.** BUILD.md is a summary/index — the per-domain files are the sources of truth.
- **Never invent content that isn't in the builder outputs.** If a builder didn't produce something, BUILD.md should say "not specified" rather than fabricate.
- **The unified todo list is the highest-value section.** Get it right. Preserve IDs. Sort correctly. Surface dependency chains.
- **Conflicts are features, not bugs.** If the plans disagree, the user needs to know. Don't paper over contradictions.

## Step 5: Tell the user what you built

Output a short summary in the chat:
- Where the dossier lives (`output/build-idea/YYYY-MM-DD-[slug]/`)
- The 6 files inside
- The top 3 P0 todos from the unified list
- Any cross-domain conflicts or open questions that need user attention first

Keep this summary under 200 words. The dossier does the real talking.

## Rules (pipeline-wide)

- Invoke each builder subagent by name explicitly (e.g., "Use the product-builder subagent to produce a build plan for this idea: ...").
- Wave 1 builders run in parallel (call in the same message). Wave 2 builders run in parallel. Wave 3 runs alone.
- Pass each wave only: (a) the full build brief and (b) **compressed 200-word summaries** of prior waves. NEVER forward full prior-builder outputs between builders.
- Keep full outputs in your own orchestrator context for the final merge only.
- Do not re-litigate viability. If the idea passed the funnel, the builders trust it and execute.
- FLAGs inherited from the evaluation are mandatory inputs. Each relevant builder must explicitly address its matching FLAG in its plan.
- All costs in euros.
- The output directory is `output/build-idea/YYYY-MM-DD-[idea-slug]/`. Never write to the old flat `output/` layout.
- Report file locations the orchestrator reads from: `output/find-ideas/*.md` and `output/evaluate-idea/*.md`.
