# Manage Plan — design spec

Date: 2026-06-24 · File: `manage-plan.html` (pricing-20-4-2026)

## Purpose
Screen where an existing **paid** user moves UP a tier. Reuses v6 tokens, credit-pool
dropdown, offer pills, and the animated "Unlimited seats" gradient. Standalone responsive
page (not the v6 absolute-positioned canvas).

## Layout
- Header: "Manage your plan" + subtitle.
- **4 tabs** = the user's current (owned) plan: `Pro Individual` · `Gold Individual` · `Pro Team` · `Gold Team`.
- Each tab shows: a **current-plan banner** (what you own) + a grid of **upgrade cards** (only valid upgrades).
- Cards use v6 plan-card style (not tables). Team cards reuse the v6 credit pool.

## Pricing
- **Individual** — Pro: $40/mo monthly, $20/mo ($240/yr) annual. Gold: $200/mo monthly, $100/mo ($1,200/yr) annual.
- **Team** — credit pool model from v6: `units = credits/5000`, `monthlyNow = floor(units * list * (1-offer))`.
  - list/unit: Pro $20, Gold $99.
  - pool/offer: 15K=10% · 25K=20% · 50K=30% · 75K=40% · 100K=50%.

## Per-tab upgrade matrix
| Tab (owned) | Banner | Upgrade cards |
|---|---|---|
| Pro Individual | Pro · Individual | Gold Individual · Pro Team · Gold Team (Recommended) |
| Gold Individual | Gold · Individual | Gold Team (Recommended) only. Pro Team is a tier downgrade and is NOT shown. |
| Pro Team | Pro Team · 15,000 credits · Unlimited seats | Buy more credits (up-size only) · Upgrade to Gold Team (Recommended). **Lock note** |
| Gold Team | Gold Team · 25,000 credits · Unlimited seats · Top tier | Buy more credits (up-size only). **Lock note** |

## Rules
- **Per-card Monthly/Annual toggle** (only on individual upgrade cards).
- Team tabs: individual cards are NOT rendered. A note replaces them:
  "Team plans can't switch back to individual."
- **Buy more credits** dropdown is up-size only: options filtered to `>= owned credits`.
- No tier downgrades: Gold can only go to Gold Team. Pro can go to Pro Team or Gold Team.
  Gold Individual → Pro Team is a downgrade and is NOT shown.

## Upgrade billing (Pro/Gold Individual → Pro Team / Gold Team)
- **No proration at checkout.** The user pays the FULL team plan price for the
  chosen credit pool (seats). Their remaining individual subscription is not
  prorated, credited, or discounted against the team price.
- **Credits carry forward.** On upgrade the pool = current plan's remaining AI
  credits + the new team plan's AI credits (both added together).
- **Renewal.** From the next billing cycle (next year) the user is billed the
  team plan price on renewal.

## Components / wiring
- `creditCard({planKey, mountId, owned, label, cta, recommended, flag})` — builds credit dropdown
  (filtered to `>= owned`), renders price via shared `monthlyNow`/`monthlyList`, animated gradient.
- `individualGoldCard()` — static card with M/A seg toggle.
- Tab switcher: show/hide `[data-tab-panel]` by owned-state key; sets `aria-selected`.
- Agentation toolbar via esm.sh module (local-link requirement).

## CTA copy
"Upgrade to Gold", "Switch to Pro Team", "Buy Gold Team", "Upgrade to Gold Team", "Buy more credits".
