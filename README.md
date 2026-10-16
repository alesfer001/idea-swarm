# Idea Swarm

A Claude Code subagent pipeline for generating, stress-testing, and planning business ideas. 21 specialized agents across 5 domains (market, product, finance, sales, legal) evaluate ideas through a kill-gate funnel with analyst/contrarian/advocate triples.

## Commands

```
/find-ideas                        Generate 3 ideas and store in .ideas/
/evaluate-idea <1|2|3|text|path>   Run an idea through the full kill-gate funnel
/build-idea <report>               Turn a validated idea into an executable build dossier
/run-gate <domain> <idea>          Run a single gate (debugging/inspection)
```

## Recommended workflow

```
/find-ideas              → generates 3 ideas, stores in .ideas/
    │
    ├─ Pick an idea (1, 2, or 3)
    │
    v
/evaluate-idea 1         → full funnel evaluation (state in .eval/)
    │
    ├─ If survived → prompted to build
    │
    v
/build-idea <report>     → executable dossier
    │
    └─ BUILD.md + product.md + legal.md + finance.md + sales.md + market.md
```

`/find-ideas` only generates — it does NOT evaluate. Each `/evaluate-idea` run stores all state on disk in `.eval/`, so it's resumable if interrupted and token-efficient (summaries read from disk, not carried in context).

## The funnel

Each gate has an analyst + contrarian pair. Contrarians return one of four verdicts:

- **PASS** — no concerns
- **FLAG** — non-fatal concerns, idea continues
- **RESHAPE** — structural flaw found; the idea pivots (new target user, revenue model, product shape, or distribution channel) and the **funnel restarts from Gate 1** with the reshaped version. Max 2 reshapes per idea — after that, further reshape attempts become FLAGs.
- **KILL** — triggers a domain advocate to challenge the kill

Advocates (activated only on KILL) return:

- **UPHOLD KILL** — idea dies
- **OVERTURN TO FLAG** — downgraded, idea continues
- **OVERTURN TO RESHAPE** — advocate provides a structural pivot, funnel restarts from Gate 1

```
/evaluate-idea (max 3 passes: original + 2 reshapes):

  ┌─────────────────────────────────────────────────────┐
  │  Market Analyst → Market Contrarian → (Advocate?)   │
  │       │                                             │
  │  Product Analyst → Product Contrarian → (Advocate?) │
  │       │                                             │
  │  Finance Analyst → Finance Contrarian → (Advocate?) │
  │       │                                             │
  │  Sales Analyst → Sales Contrarian → (Advocate?)     │
  │       │                                             │
  │  Legal Analyst → Legal Contrarian → (Advocate?)     │
  └──────────────────────┬──────────────────────────────┘
                         │
              RESHAPE? ──┤── Yes + budget remaining → restart from top
                         │── Yes + budget exhausted → forced to FLAG
                         │── No → Final Report
```

Gate order is configurable in `pipeline.md`.

## File structure

```
pipeline.md              Pipeline config (gate order, reshape budget, model)

.ideas/                  Current batch from /find-ideas (idea-1.md, idea-2.md, idea-3.md)

.eval/                   Evaluation working state (resumable)
  idea.md                Current idea (updated on reshape)
  original-idea.md       Original idea (preserved)
  state.md               Pipeline state (current gate, reshape depth, status)
  gates/                 Per-gate result files (summary + full output)
  reshapes/              Reshape records

output/
  evaluate-idea/         Final reports from /evaluate-idea runs
  build-idea/            One folder per build dossier
    YYYY-MM-DD-slug/
      BUILD.md           Master brief with unified todo list
      product.md         MVP scope, stack, timeline
      legal.md           Compliance checklist, DPAs, policies
      finance.md         12-month P&L, unit economics
      sales.md           GTM playbook, outreach templates
      market.md          Positioning, ICP, landing page outline

rejected-ideas.md        Killed ideas (prevents re-proposing)
```

## Setup

1. Clone this repo
2. Open Claude Code in the project root
3. Run `/find-ideas`

## Configuration

- **Pipeline:** Edit `pipeline.md` to add/remove/reorder gates, change reshape budget, or set default model
- **Model:** Change `model` in each agent's YAML frontmatter in `.claude/agents/` (ideation: `claude-opus-5-5`; all other agents: `claude-haiku-4-5-20251001`)
- **Agent prompts:** Edit files in `.claude/agents/` to tune evaluation criteria

## Cost

- `/find-ideas`: 1 Haiku call (ideation only)
- `/evaluate-idea` (single idea, no reshapes): ~10 Haiku calls
- `/evaluate-idea` (single idea, 1 reshape): ~20 Haiku calls
- `/evaluate-idea` (single idea, max 2 reshapes): ~30 Haiku calls (worst case)
- `/run-gate` (single gate): 2-3 Haiku calls
- `/build-idea` (single idea): ~5 Haiku calls
