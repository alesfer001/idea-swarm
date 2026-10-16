# Idea Swarm

Business idea generation and evaluation pipeline using Claude Code subagents.

## Workflow

```
/find-ideas [preset|focus] → generates 3 ideas, stores in .ideas/ (previous batch → .ideas/previous/)
                           presets live in pipeline.md, e.g. /find-ideas digital-products
/evaluate-idea 1         → normalizes idea, initializes .eval/
/run-gate market         → runs market gate
/clear                   → optional: save tokens
/run-gate product        → runs product gate
/clear
/run-gate finance → sales → legal
/clear
/finalize-evaluation     → generates final report, cleans up .eval/
/build-idea <report>     → produces executable build dossier
```

- `/run-gate --auto` chains all remaining gates then calls `/finalize-evaluation`
- `/run-gate market "some idea"` runs standalone (no `.eval/`, debugging only)
- `/clear` between gates is recommended — all state lives on disk in `.eval/`

## Rules

- **Reshape budget:** max 2 per idea. RESHAPE = structural change only (target user, revenue model, product shape, distribution channel). Pricing/messaging/scope tweaks = FLAG.
- **Business model:** no restrictions. SaaS, services, info products, templates, marketplaces, hybrid — all acceptable. Evaluate on demand, viability, and solo-executability.
- **Models:** ideation uses `claude-opus-5-5`; the other 20 agents use `claude-haiku-4-5-20251001`. Change via `model` field in agent frontmatter.

## Architecture

- **Skills:** `/.claude/skills/{find-ideas,evaluate-idea,run-gate,finalize-evaluation,build-idea}/SKILL.md`
- **Agents:** `/.claude/agents/` — 1 ideation + 5×(analyst, contrarian, advocate) + 5 builders
- **Config:** `/pipeline.md` — gate order, reshape budget, model
- **Working state:** `/.ideas/` (current batch), `/.eval/` (evaluation state — ephemeral)
- **Output:** `/output/evaluate-idea/`, `/output/build-idea/`, `/rejected-ideas.md`
- **Learnings:** `/learnings.md` — which channels have and haven't worked for the founder; update after each marketing test. `/find-ideas` passes its "What this means for new ideas" section to ideation as a hard distribution constraint, and `/run-gate` passes it to the sales gate.
