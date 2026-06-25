# Pricing & Plan Rules

Date: 2026-06-25 · Scope: `v6.html` (pricing) + `manage-plan.html` (upgrade tiers)

Single source of truth for plan pricing, upgrade paths, and billing rules.

## 1. Plans

| Plan | Type | Price | AI credits / yr |
|------|------|-------|-----------------|
| Pro  | Individual | $20/mo ($240/yr) | 5,000 |
| Gold | Individual | $100/mo ($1,200/yr) | 50,000 |
| Pro Team  | Team (credit pool) | from $60/mo | from 15,000 |
| Gold Team | Team (credit pool) | from $300/mo | from 150,000 |
| Enterprise | Custom | Talk to Sales | Custom |

- **Billing period:** all plans are annual-only (no monthly toggle).
- **Tier order:** Pro (1) < Gold (2). Team plans carry the same tier as their base.

## 2. Team pricing (credit pool)

- A team plan's **unit = one seat of its individual plan**:
  - Pro unit = 5,000 credits @ $20/mo
  - Gold unit = 50,000 credits @ $100/mo
- **Team list price = single-user price × seats.** No hidden markup.
- Seats are **unlimited**; the buyer picks a **credit pool**, which maps to a seat count.
- Pro and Gold pools are **independent** (chosen separately).
- **Volume discount ladder** by pool position (applies on top of list, shown as
  strikethrough + "you're saving $X/yr"):

| Seats | Discount | Pro pool / list / net | Gold pool / list / net |
|-------|----------|-----------------------|------------------------|
| 3  | 10% | 15,000 / $60 / $54    | 150,000 / $300 / $270 |
| 5  | 20% | 25,000 / $100 / $80   | 250,000 / $500 / $400 |
| 10 | 30% | 50,000 / $200 / $140  | 500,000 / $1,000 / $700 |
| 15 | 40% | 75,000 / $300 / $180  | 750,000 / $1,500 / $900 |
| 20 | 50% | 100,000 / $400 / $200 | 1,000,000 / $2,000 / $1,000 |

(prices /mo; net = after discount. Pro 3 users = $240×3 = $720/yr list; Gold 3 users = $1,200×3 = $3,600/yr list.)

## 3. Upgrade paths (Manage Plan)

Who can upgrade to what. **No tier downgrades.**

| Current plan | Can upgrade to |
|--------------|----------------|
| Pro Individual  | Gold Individual · Pro Team · Gold Team |
| Gold Individual | Gold Team only (Pro Team is a downgrade, hidden) |
| Pro Team  | Buy more credits (up-size) · Gold Team |
| Gold Team | Buy more credits (up-size) only (top tier) |

- **Pro → Pro Team or Gold Team.**
- **Gold → Gold Team only.**
- Once on a **team** plan, you **cannot return to an individual** plan.
- **Buy more credits** is up-size only: pool options filter to `>= current credits`.
- Enterprise is always offered as the "Talk to Sales" path.

## 4. Upgrade billing (Individual → Team)

When a Pro/Gold single user upgrades to a team plan:

- **No proration at checkout.** Pay the FULL team plan price for the chosen
  credit pool (seats). The remaining individual subscription is not prorated,
  credited, or discounted against the team price.
- **Credits carry forward.** New pool = current plan's remaining AI credits +
  the new team plan's AI credits (added together).
- **Renewal.** From the next billing cycle (next year) the user is billed the
  team plan price.

## 5. Files

- `v6.html` — public pricing page. Live:
  https://manivasakan-arch.github.io/pricing-20-4-2026/v6.html
- `manage-plan.html` — upgrade-tier screen for existing paid users. Live:
  https://manivasakan-arch.github.io/pricing-20-4-2026/manage-plan.html
- `HANDOFF-v6.md` — v6 implementation handoff.
- `docs/superpowers/specs/2026-06-24-manage-plan-design.md` — manage-plan design spec.
