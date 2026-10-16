# Pipeline Configuration

## Gates (in order)

Each gate runs an analyst + contrarian pair. On contrarian KILL, an advocate is invoked.
Add, remove, or reorder gates by editing this list. Each entry must match an agent name prefix in `.claude/agents/`.

1. market
2. product
3. finance
4. sales
5. legal

## Settings

- **Reshape budget:** 2
- **Model:** ideation uses claude-opus-5-5 (Haiku followed the ideation brief poorly: weak evidence, ignored the focus). The 20 gate and builder agents use claude-haiku-4-5-20251001.

## Presets

`/find-ideas <preset-name>` sends the preset's brief to the ideation agent instead of a one-line focus. A free-text focus (`/find-ideas "..."`) still works as before.

### digital-products

The setup that produced the MTD Landlord Ledger run (2026-10-16).

- **Searches:** 12
- **Brief:**

  > Focus: digital products that earn without the founder's ongoing time. Think templates, toolkits, spreadsheet/Notion/Airtable systems, printable or downloadable packs, info products, and small self-serve web tools (one-time purchase or cheap subscription) that a buyer finds, pays for and uses without ever talking to the founder. NOT B2B SaaS sold to businesses that needs onboarding, integrations, support or sales effort. No done-for-you work, no per-customer service.
  >
  > Evidence bar for this run (stricter than usual): demand must be shown by money already changing hands for similar products, e.g. Etsy listings with visible sales/review counts, Gumroad/Lemon Squeezy creators posting revenue, template marketplace sales, Indie Hackers/Starter Story revenue reports, or search demand for a "template"/"tool" keyword. Vendor blogs and listicles do not count as demand. Each idea needs a specific gap in what currently sells (niche, language/country, format, quality), not just "cheaper".
  >
  > The usual SOM ≥ €500K floor is relaxed for this run: a niche product earning €1–5K/month is a good outcome. Say so in Key risk.
  >
  > Cover several distinct niches/audiences; EU (incl. French-speaking) markets are welcome but not required.
