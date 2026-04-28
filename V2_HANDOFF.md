# v2.html — Engineering Handoff

> A drop-in, production-ready spec for `v2.html`. Read this **alongside** the file:
> - **HTML source:** [v2.html](https://github.com/manivasakan-arch/pricing-20-4-2026/blob/manivasakan-arch/dev-api-changes/v2.html)
> - **Live preview:** [https://manivasakan-arch.github.io/pricing-20-4-2026/v2.html](https://manivasakan-arch.github.io/pricing-20-4-2026/v2.html)

The prototype is a **single self-contained HTML file** that pulls Tailwind, Inter, Phosphor icons, and a few CDN libs. Your job is to translate it into your production stack (React/Vue/Server-rendered/etc.) while preserving the behaviors below.

---

## 1. What `v2.html` is

`v2.html` is the **canonical pricing-modal screen** with one deliberate change vs. the baseline (`index.html`):

> **The Pro card has no Monthly ↔ Annual billing-period control.** Pro is annual-only. The seg-control is preserved as a hidden anchor (`<div class="plan-seg hidden" data-pro-seg aria-hidden="true">`) so the existing JS selectors keep working and `getProPeriod()` falls through to its `'annual'` default.

Use this variant when running pricing experiments where you don't want to expose monthly billing for Pro.

The screen is a **modal-style "Pick a plan" card**, sized at 1280px width, that appears inside the product when the user hits a paywall, "Upgrade" CTA, or close-intent path. Three audience views: **Individual** and **Teams & Enterprise** (no API tab on this page).

---

## 2. Visual structure

```
┌─────────────────────────────────────────────────────────────┐
│  Title: "Pick a plan that grows with you"            [✕]    │
│                                                             │
│             [Individual]   [Teams & Enterprise]             │  ← top segmented control
│                                                             │
│   ┌────────┐    ┌───────────┐    ┌────────┐                 │
│   │ Basic  │    │   PRO ★   │    │  Gold  │                 │
│   │  $9    │    │ $20 / mo  │    │ $99/mo │                 │  ← 3 plan cards
│   │ Buy    │    │  Buy Now  │    │  Buy   │                 │
│   └────────┘    └───────────┘    └────────┘                 │
│                                                             │
│  …testimonials, sticky compare summary, feature table…      │
│  …trust strip (SOC 2 + GDPR)…                               │
│  ───────────────────────────────────────────────────────    │
│  Sticky footer: "Loved by 10M+ presenters at" + logos       │
└─────────────────────────────────────────────────────────────┘
```

- **Pro is the featured card:** `.pro-card-bg` (light beige), `border-2 border-brand-secondary` (#b8c1cc), `rounded-[4px]`, `shadow-card`, sits 21px higher and 45px taller than Basic / Gold.
- **Outer card** is white on a light page background, rounded, drop-shadow.

---

## 3. The Pro card — what's different in v2

```html
<!-- index.html (baseline) -->
<div class="plan-seg" role="tablist" aria-label="Billing period" data-pro-seg>
  <button class="plan-seg-btn" data-period="monthly" role="tab" aria-selected="false">Monthly</button>
  <button class="plan-seg-btn is-active" data-period="annual" role="tab" aria-selected="true">Annual</button>
</div>

<!-- v2.html (this file) -->
<div class="plan-seg hidden" data-pro-seg aria-hidden="true"></div>
```

**Why we kept the empty anchor:** The shared `renderPro()` / `setProPeriod()` / `getProPeriod()` functions all dereference `[data-pro-seg]`. Removing the element entirely would null-deref. The empty hidden div makes those functions still safe; `getProPeriod()` then returns `'annual'` as the default and the price stays locked.

**For your production build:** if you're rewriting `renderPro` in a real framework, you can drop both the seg-control DOM and the period-handling code paths. Pro is annual-only — render `$20/mo billed annually` directly.

---

## 4. Interactive flows

### 4.1 Top segmented control (Individual ↔ Team)
- `setTopView(target)` toggles `.hidden` on `#view-individual` / `#view-team`, syncs `aria-selected`, swaps testimonial blocks via `[data-testimonials]`, sets `body.dataset.topView`, and triggers `renderCompareSummary()`.
- **Default view: Individual.**

### 4.2 Pro card
- **Price**: static **`$20/mo billed annually`**. With the bump active (see §4.5), drops to **`$15/mo`** with a `$20` strikethrough.
- **CTA:** `Buy Now` (filled orange `pro-btn`).
- **Hover hint** (`.plan-hint`): "For those who want AI to craft polished, on-brand decks regularly".
- **Bump-state chip** (`.pro-discount-chip`) appears above the Buy Now button when the bump is active: `$60 off · ⏱ Ends in MM:SS`. Animates with a vertical bob + halo pulse.

### 4.3 Gold card (still has Monthly ↔ Annual)
- **Annual (default):** `$99/mo billed annually`.
- **Monthly:** `$199/mo billed monthly` + a green inline link "Switch to annual to save $1,200 per year". Clicking the link snaps the toggle back to annual.
- **Credits in the feature list** swap with the period:
  - Annual = **50,000**
  - Monthly = **5,000**
- Hover hint: "For those who want the best AI models to lead mission-critical decks".

### 4.4 Basic card
- Static `$9/mo billed annually`. Outline `Buy Now` button.
- Hover hint: "For those who want to make simple decks occasionally".

### 4.5 The Pro **bump** flow (close-intent → discount unlock)
This is the most important behavior to preserve in production:

```
User clicks ✕ Close
    │
    ├─ already submitted survey or bump active?  → Yes → close normally
    │
    └─ open Feedback modal (#fb-modal, window.openFeedbackModal)
            │
            ├─ Step 1: survey (sentiment smiley + improvement chip)
            │         • Submit disabled until both selected
            │
            └─ Step 2: success state
                    "Thank you. We've unlocked $60 off Pro Annual for the next one hour."
                    [ Buy Pro Annual at $60 off ]   ← calls window.applyProBump()
                            │
                            ├─ writes localStorage.proBumpStart = Date.now()
                            ├─ forces Pro → annual (via setProPeriod('annual'))
                            ├─ repaints Pro card: price = $15/mo, strike $20, chip "$60 off · 60:00"
                            ├─ starts setInterval countdown (BUMP_MS = 60 * 60 * 1000)
                            └─ on expiry: clears localStorage, re-renders Pro at $20/mo
```

State flags: `localStorage.proBumpStart`, `sessionStorage.proFeedbackSubmitted`, `sessionStorage.proBumpOffered`.

**Production checklist:**
- The bump state should live on the **server** in production (not localStorage), keyed by user. The 60-min window starts at the moment they unlock the bump.
- `window.openFeedbackModal` / `window.applyProBump` are the integration points — wire your equivalent to the actual offer endpoint.
- The `$60 off` chip animation (`pro-discount-chip` / `pro-bumped-pill` keyframes) is purely CSS — port the keyframes verbatim.

### 4.6 Team view (`#view-team`)
- Three cards: **Pro Team** (featured), **Gold Team**, **Enterprise**.
- **Shared seat-count state** (`teamState = { n, custom, hasTyped }`) — Pro Team and Gold Team mirror each other when you change the seat chip.
- Default 5 seats. Chips: 3 / 5 (with `MOST BOUGHT` badge) / 10+ (becomes a `– [number] +` stepper input on click; clamps 10–999).
- Tier rates:

| Plan | 1–4 | 5–9 | 10+ | Credits/seat/yr |
|------|-----|-----|-----|-----------------|
| Pro Team | $20 | $18 | $17 | 5,000 |
| Gold Team | $99 | $89 | $84 | 50,000 |

- CTAs: `Buy {n} Seats`. Hint copy below the CTA: dynamic savings line ("You're saving $<x> every year") when the user hits a tier discount.

### 4.7 Sticky compare summary + feature comparison table
Below the cards (`#below-views`):
- **Testimonials** (3 cards) — different copy + portraits per view (`data-testimonials="individual"` on Individual, `data-testimonials="team"` on Team).
- **Sticky mini-summary** that pins to the top of the viewport on scroll. Mirrors each card's price + Buy CTA.
- **Feature comparison table** — sections (AI / Sharing / Collaboration / Brand kit / Analytics / Integrations) with ⓘ tooltip icons.
- **Behavior:** `renderCompareSummary()` reacts to view + state:
  - **Team view:** rewrites Pro/Gold price columns to `$<n × tierRate>`, cadence reads `"N seats · billed annually"`, CTAs become `Buy Pro N Seats` / `Buy Gold N Seats`, and the credits row in the AI-Features table updates to `n × creditsPerSeat`.
  - **Individual view:** Gold summary resets to static `$99 / Buy Gold`, Pro CTA resets to `Buy Pro`, credits cells stay at `5,000 / 50,000`.
  - A `body[data-top-view="team"]` CSS rule retargets `.pricing-grid` from 4 cols → 3 cols and hides every 4n+2 child (the Basic slot in the table).

### 4.8 Trust strip + sticky footer
- Trust strip: AICPA SOC 2 SVG + GDPR SVG (`assets/trust/`) + caption "We're a SOC 2 Type II, GDPR-compliant organization."
- Sticky social-proof footer with `❤️ Loved by 10M+ presenters at` + a continuous left→right logo marquee (Microsoft, Google, Adobe, Meta, McKinsey, Amazon, Notion, EY, BCG, …). Hover pauses the animation.

---

## 5. Pricing model (canonical numbers)

Use these as the source of truth when wiring to your pricing service.

### Individual
| Plan | Annual | Monthly | Credits |
|------|--------|---------|---------|
| Basic | $9/mo billed annually | — | 1,500 / yr |
| **Pro (v2: annual-only)** | **$20/mo billed annually** ($15 with bump) | — | **5,000 / yr** |
| Gold | $99/mo billed annually | $199/mo | Annual: 50,000 / Monthly: 5,000 |

### Team (per seat, billed annually)
| Plan | 1–4 | 5–9 | 10+ | Credits / seat / yr |
|------|-----|-----|-----|---------------------|
| Pro Team | $20 | $18 | $17 | 5,000 |
| Gold Team | $99 | $89 | $84 | 50,000 |

### Bump
- **$60 off Pro Annual.** 60-minute window. One-shot per session.
- Discounted Pro price: **$15/mo billed annually** (instead of $20).
- Trigger: close-intent feedback survey completion.

---

## 6. Tech dependencies

All loaded from public CDNs in the prototype — replace with your own bundled equivalents in production.

| Library | Used for |
|---------|----------|
| Tailwind Play CDN | Utility classes (no build) |
| Google Fonts: Inter (400–900) | Body + headlines |
| Google Fonts: Hedvig Letters Serif | Testimonial blockquotes |
| Google Fonts: Roboto Condensed | Plan-seg control + MOST BOUGHT badge |
| Google Fonts: Instrument Serif | (Inherited; not actively used in v2) |
| Phosphor Icons (regular / bold / fill) | Icons throughout |
| **Optional:** React 18 + agentation@3 (esm.sh) | In-page design feedback toolbar — **strip before production** |

The page also embeds the small Individual-card illustrations as base64 data URIs (`.illus-basic`, `.illus-pro`, `.illus-gold`, `.illus-enterprise`). On-disk assets are listed in §8.

---

## 7. Production build — what to strip / replace

| Item | What to do |
|------|-----------|
| `<script type="module">…import { Agentation } from 'esm.sh/agentation@3'…</script>` + `<div id="agentation-root">` | **Remove entirely.** Design-review tool only. |
| `const DEV_RESET_ON_LOAD = true;` | Set to `false`. This is what wipes the bump + survey state on every page load while iterating. |
| Sticky-footer logo URLs (`https://www.figma.com/api/mcp/asset/...`) | **Replace with permanent CDN URLs** (the Figma asset URLs expire after 7 days). |
| Tailwind CDN | Replace with your build's Tailwind / your own utility system. |
| Google Fonts CDN | Self-host or use your existing font pipeline. |
| Phosphor Icons CDN | Same — self-host or bundle. |
| Hard-coded prices | Move to a config / pricing service. The labels are inline strings in v2.html; grep `$20`, `$99`, `5,000`, `50,000` and replace with your data. |
| `localStorage` bump persistence | Move to **server-side** offer state, keyed by user. The 60-min window should start when the offer is granted, not when the page loads. |
| `data-pro-seg` empty anchor | Optional — you can drop it once you remove the period-handling code path. |

---

## 8. Asset map

```
assets/
├── testimonials/                      # team-tab portraits
│   ├── team-marcus.png
│   ├── team-desmond.png
│   └── team-elena.png
└── trust/                             # SOC 2 + GDPR badges
    ├── soc2.svg
    └── gdpr.svg
```

Inline (base64) in `v2.html`:
- `.illus-basic`, `.illus-pro`, `.illus-gold`, `.illus-enterprise` — the small individual-card illustrations.

---

## 9. Mobile responsive behavior

A single `@media (max-width: 768px)` block in v2.html rewires the layout:

- **Outer stage:** full width, no scaling, padding `0 16px 96px` (96px bottom reserves room for the sticky social-proof footer).
- **Top segmented control:** sticky to `top: 0`, `z-index: 50`, white bg + bottom border.
- **Outer ✕ Close button:** `position: fixed; top: 10; right: 10; z-index: 60` so it stays visible while scrolling.
- **Each `#view-*` container** becomes `display: flex; flex-direction: column; gap: 12px`. `.hidden` is honored so only the active view's cards render.
- **Cards** reset to `width: 100%; max-width: 420px`, centered. **`.pro-card-bg` gets `order: -1`** so the Pro card always lands at the top of the stack on mobile.
- **Hover tooltips (`.plan-hint`)** hidden — they don't make sense on touch.
- **Comparison table block (`.compare-block`)** hidden — it doesn't scale gracefully on phones.
- **Site footer** stays `position: fixed; bottom: 0` with a tighter caption + the full-width logo marquee.
- **Feedback modal** stacks vertically with the orange offer panel (gift / tag / `$60 off Pro Annual` title) on top and the survey form below.

---

## 10. JavaScript surface area

These are the global hooks the production app needs to provide equivalents for. All live on `window` so cross-IIFE code can reach them:

| Global | Purpose |
|--------|---------|
| `window.openFeedbackModal` | Opens the close-intent feedback modal. Called from the outer ✕ Close button click handler. |
| `window.applyProBump` | Unlocks the $60-off Pro Annual bump. Called from the modal's success-step CTA. Forces Pro → annual, repaints, starts the 60-min countdown. |
| `window.renderGoldIndividual` | Repaints the Gold card based on a passed period. Used by `renderCompareSummary` when returning to the Individual view. |

Internal functions (replace with framework equivalents):

| Function | Job |
|----------|-----|
| `setTopView(target)` | Switch between Individual / Team. Drives view visibility, testimonials, body data attribute, compare summary. |
| `renderPro()` | Repaint Pro card price + chip + hint based on bump state. |
| `Gold IIFE render(period)` | Repaint Gold card price + save link based on selected period. |
| `wireTeamCard(planKey)` | Bind chip clicks + custom-stepper input for one team card; pushes its render fn into `teamRenders[]`. |
| `renderAllTeams()` | Fires every team-card render + `renderCompareSummary()`. |
| `renderCompareSummary()` | Sticky summary + feature-table reactive layer. |
| `bumpRemaining()` / `formatTimer()` / `startBumpCountdown()` | Timer utilities for the bump. |

---

## 11. Naming conventions

These hand-rolled CSS classes carry the semantic styling of v2.html. Keep the names if you can — designers will keep referring to them.

| Class | Purpose |
|-------|---------|
| `.plan-seg` / `.plan-seg-btn` | Monthly ↔ Annual seg-control (used by Gold; empty on Pro in v2). |
| `.pro-card-bg` / `.pro-btn` / `.pro-cta-wrap` | Featured-card visual treatment. |
| `.pro-discount-chip` / `.pro-bumped-pill` | Bump-state chips with bob + glow keyframes. |
| `.plan-hint` | Soft white tooltip bubble that appears on `section:hover`. |
| `.seat-chip-row` / `.seat-chip` / `.seat-chip-stepper` / `.seat-chip-input` | Team seat selector. |
| `.feature-row-label` / `.feature-row-cell` / `.feature-tip` | Comparison table primitives. |
| `.pricing-grid` | Added to the sticky summary AND every feature grid; the `body[data-top-view="team"]` rule uses it to drop the Basic column. |
| `.compare-block` | Composite class on the sticky summary + feature table sections; flipped to `display: none` on mobile. |
| `.fb-*` | Feedback modal: `.fb-card`, `.fb-survey-wrap`, `.fb-panel-left`, `.fb-right-col`, `.fb-smiley`, `.fb-chip`, `.fb-btn-primary`. |
| `.sp-wrap` / `.sp-track` / `.sp-logo` | Social-proof marquee primitives. |

---

## 12. Open questions for product

1. **Pro Monthly credits** — currently a placeholder (`X,XXX`). What's the real number when monthly billing is offered (in `index.html`, not v2)?
2. **Bump granularity** — does every user get one bump per session, per day, ever? Currently per-session (`sessionStorage.proBumpOffered`).
3. **Tablet (769–1279px)** — v2 doesn't have a `transform: scale()` fallback; the desktop layout horizontally scrolls. Confirm this is OK or we add the same pattern that's in `export-pricing.html`.
4. **`"AI Credits (annual)"` label** in the comparison table stays *"annual"* even when Gold is switched to monthly. Confirm this copy is intended.

---

## 13. References

- **Full repo README** — covers all 7 prototype pages, not just v2: [README.md](https://github.com/manivasakan-arch/pricing-20-4-2026/blob/manivasakan-arch/dev-api-changes/README.md)
- **Live preview** — [https://manivasakan-arch.github.io/pricing-20-4-2026/v2.html](https://manivasakan-arch.github.io/pricing-20-4-2026/v2.html)
- **Engineering doc (.docx)** — same content as the full README, in Word format: [Pricing-Page-Developer-Doc.docx](https://github.com/manivasakan-arch/pricing-20-4-2026/blob/manivasakan-arch/dev-api-changes/Pricing-Page-Developer-Doc.docx)

If anything in this doc drifts from what's actually in `v2.html`, the source of truth is the file itself — every behavior is implemented inline and easy to grep.
