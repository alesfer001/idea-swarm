# Idea Swarm

A Claude Code subagent pipeline for generating, stress-testing, and planning business ideas. 21 specialized agents across 5 domains (market, product, finance, sales, legal) evaluate ideas through a kill-gate funnel with analyst/contrarian/advocate triples.

## Commands

```
/find-ideas              Generate 3 ideas and run each through the full funnel
/evaluate-idea <idea>    Push your own idea through the same funnel
/build-idea <report>     Turn a validated idea into an executable build dossier
```

## Recommended workflow

```
/find-ideas
    │
    ├─ Surviving ideas get a clean re-evaluation prompt
    │
    v
/evaluate-idea <final form of surviving idea>
    │
    ├─ If survived → prompted to build
    │
    v
/build-idea <path to evaluate-idea report>
    │
    └─ Dossier: BUILD.md + product.md + legal.md + finance.md + sales.md + market.md
```

**Why the middle step?** `/find-ideas` may reshape an idea multiple times during the funnel, meaning earlier gates never saw the final form. A clean `/evaluate-idea` pass catches ideas that only survived due to reshape-context bleed.

## The funnel

Each gate has an analyst + contrarian pair. Contrarians return one of four verdicts:

- **PASS** — no concerns
- **FLAG** — non-fatal concerns, idea continues
- **RESHAPE** — the idea pivots (new target, pricing, model, etc.) and continues in its new form
- **KILL** — triggers a domain advocate to challenge the kill

Advocates (activated only on KILL) return:

- **UPHOLD KILL** — idea dies
- **OVERTURN TO FLAG** — downgraded, idea continues
- **OVERTURN TO RESHAPE** — advocate provides a pivot, idea continues reshaped

```
For each idea:

  Market Analyst → Market Contrarian → (Market Advocate if KILL)
       │
  Product Analyst → Product Contrarian → (Product Advocate if KILL)
       │
  Finance Analyst → Finance Contrarian → (Finance Advocate if KILL)
       │
  Sales Analyst → Sales Contrarian → (Sales Advocate if KILL)
       │
  Legal Analyst → Legal Contrarian → (Legal Advocate if KILL)
       │
  Final Report
```

## Output

```
output/
  find-ideas/          Reports from /find-ideas runs
  evaluate-idea/       Reports from /evaluate-idea runs
  build-idea/          One folder per build dossier
    YYYY-MM-DD-slug/
      BUILD.md         Master brief with unified todo list
      product.md       MVP scope, stack, timeline
      legal.md         Compliance checklist, DPAs, policies
      finance.md       12-month P&L, unit economics
      sales.md         GTM playbook, outreach templates
      market.md        Positioning, ICP, landing page outline

rejected-ideas.md      Killed ideas (prevents re-proposing)
```

## Setup

1. Clone this repo
2. Open Claude Code in the project root
3. Run `/find-ideas`

## Configuration

- **Model:** Change `model` in each agent's YAML frontmatter in `.claude/agents/` (default: `claude-haiku-4-5-20251001`)
- **Agent prompts:** Edit files in `.claude/agents/` to tune evaluation criteria

## Cost

- `/find-ideas` (3 ideas, typical): ~15-25 Haiku calls
- `/evaluate-idea` (single idea): ~10-12 Haiku calls
- `/build-idea` (single idea): ~5 Haiku calls
