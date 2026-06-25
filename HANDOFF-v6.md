# v6 Pricing Page — Handoff

Date: 2026-06-25 · File: `v6.html` · Branch: `feature/manage-plan`

## What it is

Self-contained pricing page prototype for Presentations AI. Single HTML file,
no build step. Tailwind (CDN), Phosphor icons, Inter font, Agentation toolbar.

- **Live:** https://manivasakan-arch.github.io/pricing-20-4-2026/v6.html
- **Local:** `python3 -m http.server 8751` in repo root, then http://127.0.0.1:8751/v6.html

## Layout

Top toggle switches two audiences:

- **Individual** — Pro and Gold cards (single user).
- **Teams & Enterprise** — Pro Team, Gold Team (credit pool), Enterprise.

Below the cards: a full feature-comparison table and a sticky social-proof footer.
A reactivation/feedback modal flow also lives on the page (dev flag resets it each load).

## Pricing model

### Individual (annual-only)

| Plan | Price | Credits/yr |
|------|-------|-----------|
| Pro  | $20/mo ($240/yr) | 5,000 |
| Gold | $100/mo ($1,200/yr) | 50,000 |

Monthly billing toggle was removed this session — all cards are annual-only.

### Team (credit pool)

Each plan's **team unit = one seat of its individual plan**, so team **list price
= single-user price × seats**:

- Pro unit = 5,000 credits @ $20/mo
- Gold unit = 50,000 credits @ $100/mo

Both use the same 5-step seat ladder (3, 5, 10, 15, 20 users) and the same
volume-discount ladder (10% / 20% / 30% / 40% / 50%). Discount applies on top of
list and shows as strikethrough + "you're saving $X/yr".

| Seats | Pro pool / list / net | Gold pool / list / net |
|-------|----------------------|------------------------|
| 3  | 15,000 / $60 / $54   | 150,000 / $300 / $270  |
| 5  | 25,000 / $100 / $80  | 250,000 / $500 / $400  |
| 10 | 50,000 / $200 / $140 | 500,000 / $1,000 / $700 |
| 15 | 75,000 / $300 / $180 | 750,000 / $1,500 / $900 |
| 20 | 100,000 / $400 / $200| 1,000,000 / $2,000 / $1,000 |

(prices /mo; net = after discount. Pro 3 users = $240×3 = $720/yr list. Gold 3 users = $1,200×3 = $3,600/yr list.)

Seats are unlimited; the dropdown picks the **credit pool**, which maps to a seat
count. Pro and Gold pools are now independent (separate state per plan).

## Key code locations (in `v6.html`)

All page logic is inline `<script>` near the bottom.

- `TEAM_PLANS` / `OFFER_LADDER` — team unit, list price, options, discounts.
- `monthlyNow(planKey, credits)` / `monthlyList(...)` — team price math.
- `offerFor(planKey, credits)` / `offerPct(...)` — discount by ladder position.
- `teamState = { credits: { pro, gold }, period }` — per-plan pool state.
- `buildCreditDropdown(planKey)` — accessible custom credit dropdown (keyboard,
  click-outside, offer pills).
- `wireTeamCard(planKey)` — renders a team card's price/credits/CTA/savings.
- `renderCompareSummary()` — syncs the sticky compare table to the active view.
- `renderGoldIndividual` / `renderPro` — individual-card price renderers.
- Feature comparison table is static HTML (search "feature-row-label").

## Changes made this session

1. Removed Monthly/Annual billing toggle from all cards (annual-only).
2. Copy polish: removed all em dashes from user-facing text (placeholder, title,
   testimonials, success label); fixed punctuation (missing `?`, double "for").
3. Removed the "Guests" row from the feature comparison table.
4. Reworked team pricing to per-plan credit pools (Pro and Gold independent).
5. Set Gold team unit to $100/seat so team list = single-user price × seats.

## Notes / follow-ups

- Sibling file `manage-plan.html` (upgrade-tier screen for existing paid users)
  reuses the same credit-pool model but has NOT been updated to the new
  single×seats Gold math — sync it if that screen ships.
- Feature table uses `—` glyphs for "not included" cells (intentional UI marker,
  not prose; left as-is despite the no-em-dash copy rule).
- Reactivation modal has `DEV_RESET_ON_LOAD = true` — flip to `false` before
  shipping if the 60-minute offer should persist across refreshes.

## Repo / deploy

- origin: `manivasakan-arch/pricing-20-4-2026` (fork) · upstream: `dhruv-saxena/pricing-20-4-2026`
- GitHub Pages serves from branch `manivasakan-arch/dev-api-changes` (path `/`).
- To update live: fast-forward that branch to the work branch and push; Pages rebuilds (~40s).
