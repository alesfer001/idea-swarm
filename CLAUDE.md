# Idea Swarm

This project is a business idea generation and evaluation pipeline using Claude Code subagents.

## How to use

The project has **three** slash commands, each for a different stage of the lifecycle: find → evaluate → build.

**1. Generate and evaluate fresh ideas:** run `/find-ideas` to start the full pipeline. It will:
1. Generate exactly 3 business ideas via the ideation agent using a **demand-first, search-first** approach (search for real pain → filter signals → build ideas backwards from demand)
2. Run each through a sequential kill-gate funnel (market → product → finance → sales → legal)
3. Each gate has an analyst + contrarian pair, plus a domain advocate that activates only on contrarian KILL
4. Contrarians can KILL, RESHAPE, FLAG, or PASS an idea
5. When a contrarian KILLs, the matching advocate challenges the KILL and can UPHOLD KILL, OVERTURN TO FLAG, or OVERTURN TO RESHAPE
6. Surviving (and possibly reshaped or debate-saved) ideas get a final report in `output/find-ideas/`
7. Killed ideas are appended to `rejected-ideas.md` so future runs don't re-propose them

**2. Evaluate a specific idea you already have:** run `/evaluate-idea <your idea description>` to skip ideation and push your own idea through the same funnel. Output goes to `output/evaluate-idea/YYYY-MM-DD-eval-[idea-slug].md`.

**3. Turn a validated idea into an executable build dossier:** run `/build-idea <path-to-report-or-raw-text>` once an idea has survived the funnel. Accepts either a path to a report in `output/find-ideas/` or `output/evaluate-idea/`, or raw idea text. If the referenced report contains multiple ideas, the orchestrator lists them with verdict + FLAG count and asks which one to build. Produces a dossier folder at `output/build-idea/YYYY-MM-DD-[idea-slug]/` containing:
- `BUILD.md` — master brief with executive summary, per-domain summaries, **unified todo list**, cross-domain conflicts, launch gate checklist, and "Monday morning" actions
- `product.md`, `legal.md`, `finance.md`, `sales.md`, `market.md` — full per-domain build plans

Unlike find/evaluate, `/build-idea` does NOT critique or gate the idea — the viability decision has already been made. Builders focus on construction: MVP scope, compliance checklist, P&L, GTM playbook, positioning.

## Recommended workflow: find → evaluate → build

The three skills are designed to chain, but **do not pipe `/find-ideas` directly into `/build-idea`**. Always put a clean `/evaluate-idea` pass in between:

1. **`/find-ideas`** — generates and filters. Surviving ideas may have been reshaped 1-3× during the funnel, which means earlier gates never saw the final form of the idea.
2. **`/evaluate-idea <final form of surviving idea>`** — re-runs the full gate funnel against the final, stabilized idea as a clean submission. No reshape context, no prior verdicts carried in. This is a consistency check: if the idea passed `/find-ideas` only because of reshape-context bleed, the clean re-evaluation will catch it.
3. **`/build-idea <evaluate-idea report>`** — only run this on an idea that has passed **both** stages.

**Why this matters:** on 2026-04-12, the TenantHub EU idea survived all 5 gates in `/find-ideas` (reshaped 3× during the run) and produced a full build dossier. A clean `/evaluate-idea` re-run on the final form **killed it at the market gate in one pass** — the contrarian caught a bundled-competition problem that none of the reshape-context gates had seen. The find→evaluate→build sequence would have caught this before the build-idea run.

**Known framework limitation:** reshape verdicts patch the idea forward rather than rewinding to gate 1. A reshaped idea can exit `/find-ideas` without any gate having evaluated the final version. The find→evaluate→build workflow is the current mitigation; a deeper fix (rewind-on-reshape) is tracked as future work.

## Business model constraint

There is **no restriction on how hands-on the business is**. Any revenue model is acceptable: SaaS, productized services, consulting-plus-tool, info products, templates, agency models, one-time purchases, marketplaces (if the cold-start problem is addressed), or hybrid models. The pipeline evaluates ideas purely on demand, viability, and solo-executability — not on how active the day-to-day operation is.

## Architecture

- `/.claude/skills/find-ideas/SKILL.md` — generate-and-evaluate orchestrator
- `/.claude/skills/evaluate-idea/SKILL.md` — evaluate-a-specific-idea orchestrator (skips ideation)
- `/.claude/skills/build-idea/SKILL.md` — turn a validated idea into a build dossier
- `/.claude/agents/` — **21 subagents** (1 ideation + 5 domain analysts + 5 contrarians + 5 advocates + 5 builders)
- `/rejected-ideas.md` — garbage bin of ideas killed by the pipeline; the ideation agent reads this before generating new ideas
- `/output/find-ideas/` — reports from `/find-ideas` runs
- `/output/evaluate-idea/` — reports from `/evaluate-idea` runs
- `/output/build-idea/YYYY-MM-DD-[idea-slug]/` — one folder per build, containing `BUILD.md` + 5 per-domain files

## Agents

### Evaluation agents (used by `/find-ideas` and `/evaluate-idea`)

| Order | Analyst | Contrarian | Advocate (KILL only) | Gate Focus |
|-------|---------|------------|----------------------|------------|
| 1 | ideation | (generates ideas) | — | Idea generation (skipped by `/evaluate-idea`) |
| 2 | market-analyst | market-contrarian | market-advocate | Demand, competition, timing |
| 3 | product-analyst | product-contrarian | product-advocate | Feasibility, dev effort, maintenance |
| 4 | finance-analyst | finance-contrarian | finance-advocate | Unit economics, costs, projections |
| 5 | sales-analyst | sales-contrarian | sales-advocate | GTM, channels, customer acquisition |
| 6 | legal-analyst | legal-contrarian | legal-advocate | GDPR, IP, platform rules, EU compliance |

### Build agents (used by `/build-idea`)

| Wave | Builder | Focus |
|------|---------|-------|
| 1 (parallel) | product-builder | MVP scope, stack, week-by-week plan, pitfalls, exit criteria |
| 1 (parallel) | legal-builder | Compliance checklist, DPAs, policies, launch-blocking docs |
| 2 (parallel) | finance-builder | 12-month P&L, break-even, cost ceilings, pricing experiments |
| 2 (parallel) | sales-builder | First-10-customers playbook, outreach templates, weekly cadence |
| 3 | market-builder | Positioning, ICP, differentiators, messaging, landing page outline |

Builders are **construction planners, not gates**. They assume the idea has already survived the viability funnel. Any FLAGs raised during evaluation are carried forward as mandatory inputs to the matching builder (e.g., a legal FLAG must be explicitly addressed in the legal build plan).

## Contrarian verdicts

Every contrarian can return one of four verdicts:

- **KILL** — fatal flaw. Triggers the matching advocate to challenge the kill.
- **RESHAPE** — the underlying demand is real but the current product shape, audience, pricing, or GTM is wrong. The contrarian rewrites the idea with a concrete, specific pivot. The reshaped version **replaces** the original for all downstream agents.
- **FLAG** — non-fatal concerns worth noting. Continue the pipeline.
- **PASS** — no concerns at this gate.

## Advocate verdicts (new)

Advocates ONLY activate when their matching contrarian returns KILL. They receive the idea, analyst findings, and contrarian's KILL reasoning, and return one of:

- **UPHOLD KILL** — the contrarian is right, the kill stands. Idea dies.
- **OVERTURN TO FLAG** — the contrarian's concerns are real but not fatal. Idea continues with a FLAG.
- **OVERTURN TO RESHAPE** — the contrarian identified real problems, but a specific pivot addresses them. Idea continues with the advocate's reshaped version.

Advocates are cost-efficient: most runs trigger only 0-3 advocate calls because most gates return FLAG/PASS/RESHAPE rather than KILL.

**The pipeline evolves ideas, not just filters them.** Reshapes (from contrarians OR advocates) are NOT vague advice — they're specific, nameable changes to target audience, pricing, feature set, positioning, distribution channel, or business model.

### Significantly reshaped ideas should be re-evaluated

If an idea was reshaped one or more times, each downstream agent only saw the reshaped version from the reshape point onward. When the final version diverges significantly from the original, the final report flags it and recommends running `/evaluate-idea [final version]` to validate the reshaped idea through a clean run.

## Analysts and reshaped ideas

All 5 analysts carry a note at the top of their instructions: they evaluate the idea **as presented to them** and do not reference prior versions. This keeps downstream analysis clean when an upstream contrarian or advocate has reshaped the idea.

## Ideation memory bleed protection

The ideation agent is explicitly instructed to **ignore** any information about the user's personal interests, hobbies, tech stack, or background that may be present in conversation context or memory. Ideas are generated from market demand signals, not from the user's known skills.

## Rejected ideas garbage bin

After each run, the orchestrator appends every KILLED idea (contrarian KILL + advocate UPHOLD KILL) to `rejected-ideas.md` in the project root. Before generating new ideas, the ideation agent reads this file and avoids proposing substantially similar concepts (same target user + same core value prop). Entries older than 3 months can be reconsidered.

## Model

- **All 21 agents** use `claude-haiku-4-5-20251001` (Haiku 4.5) — cost-efficient evaluation and planning against defined criteria. Ideation needs web search but works well on Haiku with the structured demand-first process.

To change, edit the `model` field in each agent's frontmatter.

## Cost estimate

**Evaluation (`/find-ideas`, `/evaluate-idea`):**
- Per idea (killed early, no advocate overturn): 3 Haiku calls (analyst + contrarian + advocate)
- Per idea (full funnel, no KILLs): 10 Haiku calls (5 analyst + 5 contrarian)
- Per idea (full funnel with 1-2 advocate debates): 11-12 Haiku calls
- Ideation: 1 Haiku call with web search
- Typical `/find-ideas` run (3 ideas, 1-2 killed mid-funnel, 0-2 advocate debates): ~15-25 Haiku calls
- Inter-agent context compressed to ~150 words per gate to minimize tokens
- Advocates are only invoked on KILL, keeping average cost low

**Build (`/build-idea`):**
- 5 builder calls (product, legal, finance, sales, market) — one per domain, no contrarians
- Inter-wave context compressed to ~200 words per prior builder summary
- Typical `/build-idea` run: ~5 Haiku calls
