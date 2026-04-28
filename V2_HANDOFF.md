# v2.html — Engineering Handoff (detailed)

> Production-grade specification for `v2.html`. Read it **alongside** the file:
> - **HTML source:** [v2.html](https://github.com/manivasakan-arch/pricing-20-4-2026/blob/manivasakan-arch/dev-api-changes/v2.html)
> - **Live preview:** [https://manivasakan-arch.github.io/pricing-20-4-2026/v2.html](https://manivasakan-arch.github.io/pricing-20-4-2026/v2.html)
> - **Repo README** (whole project): [README.md](https://github.com/manivasakan-arch/pricing-20-4-2026/blob/manivasakan-arch/dev-api-changes/README.md)

The prototype is a **single, self-contained HTML file** with inline `<style>` and `<script>` blocks. Tailwind, Inter, Phosphor icons and (optionally) React + Agentation are loaded from public CDNs. Your job is to translate the behaviors below into your production stack while preserving the interaction model, animations, copy, pricing, and accessibility.

---

## Table of contents

1. [What v2.html is](#1-what-v2html-is)
2. [Visual structure](#2-visual-structure)
3. [Pricing model & numbers](#3-pricing-model--numbers)
4. [Top tab control & view switching](#4-top-tab-control--view-switching)
5. [Individual view — per-card behavior](#5-individual-view--per-card-behavior)
   - 5.1 Basic
   - 5.2 Pro (annual-only in v2)
   - 5.3 Gold (Monthly ↔ Annual + "Switch to annual to save $1,200" link)
   - 5.4 Plan hover hints (`.plan-hint`)
6. [Teams & Enterprise view](#6-teams--enterprise-view)
   - 6.1 Shared seat-count state (`teamState`) — changes on **any** card mirror to all
   - 6.2 Tiered per-seat pricing
   - 6.3 The custom "10+" stepper
   - 6.4 Savings hint copy
   - 6.5 Enterprise card
7. [Sticky compare summary + feature comparison table](#7-sticky-compare-summary--feature-comparison-table)
8. [Close-intent feedback flow + Pro Annual bump](#8-close-intent-feedback-flow--pro-annual-bump)
9. [All animations & keyframes](#9-all-animations--keyframes)
10. [Trust strip & sticky social-proof footer](#10-trust-strip--sticky-social-proof-footer)
11. [Mobile responsive behavior (≤768px)](#11-mobile-responsive-behavior-768px)
12. [JavaScript surface area & global hooks](#12-javascript-surface-area--global-hooks)
13. [CSS classes & design tokens](#13-css-classes--design-tokens)
14. [Asset map](#14-asset-map)
15. [Production checklist — what to strip / replace](#15-production-checklist--what-to-strip--replace)
16. [Open questions for product](#16-open-questions-for-product)

---

## 1. What v2.html is

`v2.html` is the **canonical pricing-modal screen** that appears inside the product when a user hits a paywall, clicks "Upgrade", or attempts to close certain flows. It contains four deliberate diffs versus the baseline (`index.html`):

1. **Pro card has no Monthly ↔ Annual billing-period control.** Pro is annual-only. The seg-control DOM (`<div class="plan-seg" data-pro-seg>`) is preserved as a **hidden anchor** so the existing `renderPro()` / `getProPeriod()` selectors keep resolving and `getProPeriod()` falls through to its `'annual'` default.
2. **`Export to PowerPoint and Google Slides`** bullet is removed from the Basic feature list and the Pro Team feature list.
3. **The bumped `$60 off · Ends in MM:SS` pill is suppressed on the Teams tab.** Both `renderPro()` and `renderCompareSummary()` check `body.dataset.topView` before painting `#sum-pro-pill-slot`. The bump applies only to Pro Annual on Individual.
4. **Closing the modal from the Teams tab snaps the page back to Individual first** so the bump unlock lands in the right (Pro Annual) context behind the modal.

Two top-level audience tabs: **Individual** and **Teams & Enterprise**. (No API tab on this page.)

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
│   │  $9    │    │ $20 / mo  │    │ $99/mo │                 │
│   │ Buy    │    │  Buy Now  │    │  Buy   │                 │  ← 3 plan cards (Individual)
│   └────────┘    └───────────┘    └────────┘                 │
│                                                             │
│  …testimonials, sticky compare summary, feature table…      │
│  …trust strip (SOC 2 + GDPR)…                               │
│  ───────────────────────────────────────────────────────    │
│  Sticky footer: "Loved by 10M+ presenters at" + logos       │
└─────────────────────────────────────────────────────────────┘
```

Outer dimensions:
- `.relative.mx-auto` outer card: **1280px wide**, white background, modal-style elevation.
- View row (`#view-individual` / `#view-team`): absolutely positioned at `top:176px; height:624px` with cards inside laid out absolutely.
- Pro card sits **21px higher** than its neighbors (`top:-21px; height:624px`) and wears the featured treatment (`.pro-card-bg`, `border-2 border-brand-secondary`, `rounded-[4px]`, `shadow-card`). Inner padding is `pt-[45px]` to compensate for the 21px lift.
- Basic and Gold cards are `top:0; height:579px`; Basic has only left-rounded corners, Gold has only right-rounded corners, so the three together visually adjoin into one shape with Pro pushing forward.

---

## 3. Pricing model & numbers

### Individual

| Plan | Annual | Monthly | Credits |
|------|--------|---------|---------|
| Basic | $9/mo billed annually | — | 1,500 / yr |
| **Pro (v2: annual-only)** | **$20/mo billed annually** ($15 with bump) | — | 5,000 / yr |
| Gold | $99/mo billed annually | $199/mo | Annual: 50,000 / Monthly: 5,000 |

### Team (per seat, billed annually)

| Plan | 1–4 | 5–9 | 10+ | Credits / seat / yr |
|------|-----|-----|-----|---------------------|
| Pro Team | $20 | $18 | $17 | 5,000 |
| Gold Team | $99 | $89 | $84 | 50,000 |

### Bump (close-intent unlock)
- **$60 off Pro Annual.** 60-minute window. One per session.
- Discounted Pro: **$15/mo billed annually** (vs $20).
- Trigger: feedback survey completion via the close-intent flow.

---

## 4. Top tab control & view switching

Element: `[data-top-seg]`. Buttons are `.top-view-seg-btn` with `data-view="individual" | "team"`. Initial active button: Individual.

### `setTopView(target)` — single source of truth
1. Toggles `.is-active` and `aria-selected` on each seg-button.
2. Toggles `.hidden` on `#view-individual` and `#view-team`. `.hidden` is Tailwind `display:none`.
3. Swaps testimonial blocks via `[data-testimonials="individual"]` ↔ `[data-testimonials="team"]`.
4. Sets `document.body.dataset.topView = target`. CSS rules then react:
   - `body[data-top-view="team"] .pricing-grid { grid-template-columns: 320px 1fr 1fr !important; }` — feature comparison drops the Basic column on team view.
   - `body[data-top-view="team"] .pricing-grid > :nth-child(4n+2) { display: none; }` — every 4n+2 grid child (the Basic data cell in each row) is hidden.
5. Calls `renderCompareSummary()` so the sticky summary + feature-table react.

### Initial render
At DOM ready, `setTopView('individual')` runs once explicitly so the Individual view is always the first thing the user sees, even if a stale `is-active` state survived a soft reload.

---

## 5. Individual view — per-card behavior

### 5.1 Basic

- Static card. Price `$9/mo` billed annually. Outline `Buy Now` button.
- Features list (top → bottom):
  - 1,500 Credits (custom orange star SVG)
  - 300 slides (`ph ph-presentation-chart`)
  - Basic AI model and agent (`ph ph-star-four`)
  - *(no longer shows "Export to PowerPoint and Google Slides" — removed in v2)*
- Hover hint (`.plan-hint`): "For those who want to make simple decks occasionally"

### 5.2 Pro (annual-only in v2)

The featured center card. **No billing-period seg-control** — it was removed for v2. The wrapping `<div class="plan-seg" data-pro-seg>` element is **kept as an empty hidden anchor** so the JS selectors still resolve.

**Default state** (no bump active):
- Price block (`[data-pro-price]`): `$20/mo` + cadence "billed annually".
- CTA: filled orange `pro-btn` "Buy Now" inside `.pro-cta-wrap` (`#pro-cta-wrap`).
- Hint paragraph (`#pro-hint`): empty + `.hidden`.
- `.pro-discount-chip` (`#pro-discount-chip`) is `.hidden`.

**Bumped state** (after the close-intent survey unlocks the bump):
- Price block shows the strikethrough: `<p class="line-through">$20</p>` next to the new big `$15` price + "billed annually".
- `.pro-discount-chip` becomes visible above the Buy Now button: `$60 off · ⏱ Ends in MM:SS`. Animates with vertical bob + halo pulse (see §9).
- `localStorage.proBumpStart` carries the timestamp. `setInterval(tick, 1000)` updates the chip's `[data-pro-timer]` text every second using `formatTimer(ms) → "MM:SS"`.

**Sticky summary mirror:** the Pro column of the compare summary at the bottom shows the same `$15` (with `$20` strikethrough) and embeds a `.pro-bumped-pill` with the matching timer in `#sum-pro-pill-slot`. **This pill is suppressed on the Team tab** (see §1.3).

Hover hint: "For those who want AI to craft polished, on-brand decks regularly".

### 5.3 Gold (Monthly ↔ Annual + "Switch to annual to save $1,200 per year" link)

The Gold card **does** have a Monthly ↔ Annual seg-control: `[data-gold-seg]` with two `.plan-seg-btn` buttons (`data-period="monthly" | "annual"`). Initial active: `annual`.

**Annual state (default):**
- Price block (`[data-gold-price]`):
  ```
  $99 /mo
  billed annually
  ```
- Save line (`[data-gold-save]`): empty (`.hidden`).

**Monthly state:**
- Price block:
  ```
  $199 /mo
  billed monthly
  ```
- Save line shows a green link directly below the Buy Now button:
  > **"Switch to annual to save $1,200 per year"**

  Markup:
  ```html
  <button type="button"
          class="underline underline-offset-[3px] decoration-from-font
                 text-[#16a34a] hover:text-[#15803d]"
          data-switch-yearly>
    Switch to annual to save $1,200 per year
  </button>
  ```
  Click handler: snaps the seg-control back to **Annual** (`setGoldPeriod('annual')`) and re-runs `render('annual')`. The page transitions smoothly (`.fade-in` keyframe on the price block re-injection — see §9).

**Credits in the Gold feature list flip with the period:**
- Annual → **50,000 Credits**
- Monthly → **5,000 Credits**
- The number lives in `<span id="gold-ind-credits">…</span>`. `window.renderGoldIndividual` is exposed globally so `renderCompareSummary` can repaint it on view switches.

Hover hint: "For those who want the best AI models to lead mission-critical decks".

### 5.4 Plan hover hints (`.plan-hint`)

A soft-light tooltip bubble rendered above each card and triggered by `section:hover > .plan-hint`. White card with `1px #e5e5e5` border, 12px medium grey text, downward arrow notch on the bottom edge, 0.18s fade-in via opacity + 2px translateY.

Hidden on mobile (touch UX).

---

## 6. Teams & Enterprise view

Three cards: **Pro Team** (featured, like Pro on Individual), **Gold Team**, **Enterprise**.

### 6.1 Shared seat-count state — *changing seats on any card mirrors to ALL cards*

Both team cards share a single `teamState` object:
```js
const teamState = { n: 5, custom: false, hasTyped: false };
```

Where:
- `n` is the current seat count (default 5, clamped 1–999).
- `custom` is `true` once the user picks the "10+" chip (which morphs into a stepper input).
- `hasTyped` is `true` once the user has typed into the stepper input (used to gate the "10+" → 10 default snap).

**Both cards register their render functions in a shared `teamRenders[]` array.** On any change:
1. The chip click handler updates `teamState.n` and `teamState.custom`.
2. `renderAllTeams()` runs every render in the array, then calls `renderCompareSummary()`.
3. **Result:** changing the chip on Pro Team updates Pro Team **and** Gold Team **and** the sticky compare summary in lockstep — totals, "MOST BOUGHT" badge position, savings hint, credits in the feature list, and the Buy CTA copy all update simultaneously.

### 6.2 Tiered per-seat pricing

```js
const TEAM_PLANS = {
  pro:  { list: 20, tiers: [[4,20],[9,18],[Infinity,17]], creditsPerSeat: 5000  },
  gold: { list: 99, tiers: [[4,99],[9,89],[Infinity,84]], creditsPerSeat: 50000 },
};
```

`rateFor(plan, n)` returns the per-seat rate for the seat count. The card price block renders `n × rate` as the total per month. When `rate < list`, a strikethrough of the list-price total is shown above the discounted total:

```
$5,000  ← strike, shows when discount tier reached
 $4,400 /mo
5 seats · billed annually
```

The credits row in the feature list shows `n × creditsPerSeat`.

### 6.3 The custom "10+" stepper

The third chip in each row has class `seat-chip custom` and `data-value="custom"`. On click:
- `teamState.custom = true`.
- The `<span class="custom-label">10+</span>` is hidden (`display:none`).
- A `.seat-chip-stepper` div is appended **inside** the chip, containing:
  ```html
  <button class="step" data-dec aria-label="Decrease seats">– icon</button>
  <input type="number" class="seat-chip-input" min="10" max="999" aria-label="Seat count">
  <button class="step" data-inc aria-label="Increase seats">+ icon</button>
  ```
- `decBtn.disabled = (n <= 10)` so the user can't decrement below 10 from the custom state.
- The stepper has **per-card DOM** but they share state: typing into one input updates `teamState.n` and re-renders every team card. To prevent a double-render race when the user is typing fast, the renderer skips writing back to the input the user is currently focused on (`typingIn` guard).

When the user clicks a non-custom chip after using the custom stepper, the stepper is torn down (`.remove()`) and the `10+` label is restored. `teamState.custom = false; teamState.hasTyped = false`.

Default snap: clicking custom for the first time sets `teamState.n = 10` (since `hasTyped` is false). After that, the typed value persists across chip toggles within the session.

### 6.4 Savings hint copy

Below each team card's Buy CTA, `#pro-team-hint` / `#gold-team-hint` show a dynamic line:

- **Tier discount reached** (`rate < list`):
  ```
  ✓ You're saving $<savings> every year
  ```
  where `savings = n × (list - rate) × 12`. Green `text-success-fg` color, ph-fill ph-check-circle icon.

- **No discount yet** (n <= 4 on Pro / Gold):
  ```
  Volume pricing starts at 5 seats
  ```
  Muted `text-ink-tertiary`.

### 6.5 Enterprise card

Static. Custom `Contact sales` CTA. Dedicated subtitle "tailored for your organization" (no per-seat math). Receives no updates from `teamState`.

---

## 7. Sticky compare summary + feature comparison table

Below the card row sits `#below-views`, which contains:
1. **Testimonials** (3 cards) — different copy + portraits per view (`data-testimonials="individual"` shown on Individual, `data-testimonials="team"` on Team).
2. **Sticky mini-summary** — `position: sticky; top: 0` so it pins as the user scrolls. Mirrors each card's price + Buy CTA. `#sum-pro-pill-slot` carries the `.pro-bumped-pill` mirror when the bump is active *and* the view is Individual.
3. **Feature comparison table** — sections (AI / Sharing / Collaboration / Brand kit / Analytics / Integrations) with ⓘ tooltip icons.

### `renderCompareSummary()` — the reactive layer

```js
function renderCompareSummary() {
  const view = document.body.dataset.topView || 'individual';

  if (view === 'team') {
    // Pro/Gold price columns become "$<n × tierRate>".
    // Cadence becomes "N seats · billed annually".
    // CTAs become "Buy Pro N Seats" / "Buy Gold N Seats".
    // Credits row in the AI table becomes n × creditsPerSeat.
    // Suppress #sum-pro-pill-slot (clear innerHTML).
  } else {
    // Gold summary resets to static $99 / Buy Gold.
    // Pro CTA resets to "Buy Pro".
    // Credits cells stay at 5,000 / 50,000.
    // renderPro() repaints Pro column; window.renderGoldIndividual(period) repaints Gold.
  }
}
```

The `body[data-top-view="team"]` CSS rule retargets every `.pricing-grid` from 4 columns to 3 and hides every 4n+2 child (the Basic slot in both the sticky summary and each feature row).

---

## 8. Close-intent feedback flow + Pro Annual bump

### Trigger: outer ✕ Close click

```
User clicks ✕ Close
    │
    ├─ already submitted survey (sessionStorage.proFeedbackSubmitted === 'true')? → Yes → close normally
    ├─ bump still active (bumpRemaining() > 0)?                                     → Yes → close normally
    │
    ├─ on Teams tab? → setTopView('individual')
    │                  (pre-flip BEFORE opening the modal so the Pro Annual
    │                   context is visible behind the modal when it appears)
    │
    └─ window.openFeedbackModal()
            │
            ├─ Step 1 — survey (#fb-step-form):
            │     • sentiment smileys (3 options: bad / ok / great) — `.fb-smiley`
            │     • improve chips (7 options: design / content / templates / speed /
            │       collaboration / export / other) — `.fb-chip`
            │     • Submit ("Submit to unlock discount") disabled until BOTH are picked
            │     • Submit click runs the lock-pop animation (see §9), then transitions to step 2
            │
            └─ Step 2 — success (#fb-step-success):
                    "Thank you. We've unlocked $60 off Pro Annual for the next one hour."
                    [ Buy Pro Annual at $60 off ]   ← #fb-claim → window.applyProBump()
                            │
                            ├─ writes localStorage.proBumpStart = Date.now()
                            ├─ sessionStorage.proFeedbackSubmitted = 'true'
                            ├─ forces Pro → annual (setProPeriod('annual'))
                            ├─ repaints Pro: price = $15/mo (strike $20), chip "$60 off · 60:00"
                            ├─ starts setInterval(tick, 1000) — BUMP_MS = 60 * 60 * 1000
                            └─ on expiry: clears localStorage, re-renders Pro at $20/mo
```

### State flags
| Storage | Key | Purpose |
|---------|-----|---------|
| localStorage | `proBumpStart` | Timestamp when the bump unlocked. `bumpRemaining()` returns `BUMP_MS - (Date.now() - start)`. |
| localStorage | `proFlowVersion` | Migration flag — clears stale v1 state on first load, then sticks at `'v2'`. |
| sessionStorage | `proFeedbackSubmitted` | True after Submit. Suppresses the close-intent modal for the rest of the tab session. |
| sessionStorage | `proBumpOffered` | Reserved (currently unused for gating). |

### Dev override
`const DEV_RESET_ON_LOAD = true;` near the top of the script wipes `proBumpStart` / `proFlowVersion` / both session flags on every load so the prototype always starts clean. **Flip to `false` in production** so the bump actually persists across reloads inside its 60-minute window.

### Pill suppression on Team
- `renderPro()` only paints `#sum-pro-pill-slot` when `body.dataset.topView !== 'team'`. Even if the bump is active and the user is on Team, the slot stays empty.
- `renderCompareSummary()` clears the slot in its team branch as a belt-and-braces.

---

## 9. All animations & keyframes

| Class | Keyframe | Duration | Where used |
|-------|----------|----------|------------|
| `.fade-in` | `fadeIn` (opacity 0→1, translateY 2→0) | 0.18s ease | New price block injection on Gold period switch + Pro card price re-renders. |
| `.pro-discount-chip` | `pro-chip-bob` (translateY 0 → -3px → 0) | 2.6s ease-in-out infinite | Vertical bob on the Pro card discount chip. |
| `.pro-discount-chip` | `pro-chip-glow` (box-shadow with 0 → 9px outward orange ring fading 0.55 alpha → 0) | 2.2s ease-in-out infinite | Halo pulse on the Pro card discount chip. |
| `.pro-bumped-pill` | `pro-pill-bob` (translateY 0 → -3px → 0) | 2.6s ease-in-out infinite | Vertical bob on the sticky-summary mirror pill. |
| `.pro-bumped-pill` | `pro-pill-glow` (box-shadow with 0 → 9px outward orange ring; **6 16 dropshadow** instead of 4 10 — bigger drop) | 2.2s ease-in-out infinite | Halo pulse on the sticky-summary mirror pill. |
| `.sp-track` (footer logos) | `sp-marquee` (translateX 0 → -50%) | 40s linear infinite | Continuous left→right marquee of brand logos. Hover pauses. |
| `.fb-gift` (feedback modal 🎁) | `fb-gift-bounce` (translateY 0 → -8px → 0) | 1.8s ease-in-out infinite | Gift emoji bounce. |
| `.fb-panel-sub .arrow` | `fb-arrow-pulse` (translateX 0 → 5px → 0, opacity 0.8 → 1 → 0.8) | 1.4s ease-in-out infinite | Arrow nudge on the modal panel sub-line. |
| `.fb-conf` (confetti pieces) | `fb-float-drift` (random translate + rotate) | 4.2–6s ease-in-out infinite, staggered delay | Floating confetti in the orange offer panel. |
| `.fb-lock` | `lock-pop` (`cubic-bezier(0.34, 1.35, 0.64, 1)` scale + rotate to lock-open) | 0.55s | Lock-icon pop when the Submit button finishes. |
| `.fb-btn-primary::after` | `cta-shine` (linear-gradient sweep across the button) | 0.75s linear once | Shine sweep on the primary CTA after activation. |
| `.fb-cta-badge` | `fb-cta-badge-pulse` (scale + box-shadow) | 1.8s ease-in-out infinite | Pulse on the success-state "Ends in MM:SS" badge. |
| `#page-dim` overlay | `pro-spot` (radial spotlight on the Pro card) | 1.2s ease-in-out infinite | After the user dismisses the success state without claiming. |
| `.plan-hint` | (transition only — opacity 0→1, translateY 2→0) | 0.18s ease | Hover bubble above each plan card. |

Mobile-specific:
- The plan-hint transitions are **disabled on mobile** (`.plan-hint { display: none !important }` inside the `@media (max-width: 768px)` block).

---

## 10. Trust strip & sticky social-proof footer

### Trust strip
Centred row above the bottom spacer:
```
[soc2.svg 60×60] [gdpr.svg 60×60]   We're a SOC 2 Type II, GDPR-compliant organization.
```
Both badges are SVG files in `assets/trust/`. Caption is 14px medium `#525252`.

### Sticky social-proof footer
A `<aside class="site-footer">` pinned to the viewport bottom (`position: fixed; bottom: 0`). Inside:
```
❤️ Loved by 10M+ presenters at  [Microsoft] [Google] [Adobe] [Meta] [McKinsey] [Amazon] [Notion] [EY] [BCG]
```

The marquee track (`.sp-track`) holds two copies of the logo set so the `translateX(0 → -50%)` loop is seamless. Hover anywhere on `.sp-wrap` pauses the animation. Two soft white gradients fade the left + right edges so logos enter/exit cleanly.

---

## 11. Mobile responsive behavior (≤768px)

The `@media (max-width: 768px)` block rewires the layout:

- **Outer stage** becomes full-width with `padding: 0 16px 96px` (96px bottom reserves space for the sticky social-proof footer).
- **Top segmented control** is sticky to `top: 0`, `z-index: 50`, white bg + bottom border.
- **Outer ✕ Close button** is `position: fixed; top: 10px; right: 10px; z-index: 60` so it stays visible while scrolling.
- **Each `#view-*`** becomes `display: flex; flex-direction: column; gap: 12px`. `.hidden` is honored so only the active view's cards render.
- **Cards** reset to `position: static; width: 100%; max-width: 420px`, centered. **`.pro-card-bg` (Pro) gets `order: -1`** so the featured card is always on top of the stack.
- **Hover tooltips (`.plan-hint`)** are `display: none` — touch UX.
- **Comparison table block (`.compare-block`)** is `display: none`.
- **Site footer** stays `position: fixed; bottom: 0` with a tighter caption + the full-width logo marquee.
- **Feedback modal** stacks vertically: orange offer panel (gift / tag / "$60 off Pro Annual" title) on top, survey form below; gift emoji forced to `display: block` so the tag drops to its own line.

---

## 12. JavaScript surface area & global hooks

These are the **integration points** for production. Live on `window` so cross-IIFE code can reach them:

| Global | Purpose |
|--------|---------|
| `window.openFeedbackModal()` | Opens the close-intent feedback modal. Wire your equivalent to the close button + close-intent triggers. |
| `window.applyProBump()` | Unlocks the $60-off Pro Annual bump. Forces Pro → annual, repaints, starts the 60-min countdown. Wire to the actual offer endpoint. |
| `window.renderGoldIndividual(period)` | Repaints the Gold card based on a passed period (`'monthly'` or `'annual'`). Used by `renderCompareSummary` when returning to Individual. |

Internal functions (replace with framework equivalents):

| Function | Job |
|----------|-----|
| `setTopView(target)` | Switch tabs. Drives view visibility, testimonials, body data attribute, compare summary, and pre-flips on the close-intent path. |
| `renderPro()` | Repaint Pro card price + chip + hint based on bump state. Reads view from `body.dataset.topView`. |
| `getProPeriod()` / `setProPeriod(period)` | Pro period accessor / setter. Falls through to `'annual'` since v2 has no Pro seg-buttons. |
| `Gold IIFE render(period)` | Repaint Gold card price + save link based on selected period. |
| `wireTeamCard(planKey)` | Bind chip clicks + custom-stepper input for one team card; pushes its render into `teamRenders[]`. |
| `renderAllTeams()` | Run every team-card render + `renderCompareSummary()`. |
| `renderCompareSummary()` | Sticky summary + feature-table reactive layer. |
| `bumpRemaining()` / `formatTimer()` / `startBumpCountdown()` | Bump timer utilities. |

---

## 13. CSS classes & design tokens

### Tokens (declared inline in `tailwind.config`)
- `brand.500 = #ff5500` (primary brand orange) + `brand.50/100/600/700` shades
- `brand-secondary = #b8c1cc` (neutral edge for the featured card)
- `ink.{primary, secondary, tertiary} = #171717 / #525252 / #a3a3a3`
- `success.{bg, fg} = #dcfce7 / #16a34a` (green save link)
- `border.{primary, secondary, tertiary}` for card edges

### Hand-rolled component classes
| Class | Job |
|-------|-----|
| `.plan-seg` / `.plan-seg-btn` / `.plan-seg-btn.is-active` | Monthly ↔ Annual seg-control (Roboto Condensed, 11px upper-case). |
| `.pro-card-bg` / `.pro-btn` / `.pro-cta-wrap` | Featured-card visual treatment. |
| `.pro-discount-chip` / `.pro-bumped-pill` | Bump-state chips with bob + glow keyframes. Top-arrow notch via `::after`/`::before`. |
| `.plan-hint` | Soft white tooltip bubble above each card on `section:hover`. |
| `.seat-chip-row` / `.seat-chip` / `.seat-chip-stepper` / `.seat-chip-input` / `.seat-chip-badge` | Team seat selector. `.seat-chip.active` style + `MOST BOUGHT` badge. |
| `.feature-row-label` / `.feature-row-cell` / `.feature-tip` | Comparison table primitives. |
| `.pricing-grid` | Added to the sticky summary AND every feature grid; the `body[data-top-view="team"]` rule uses it to drop the Basic column. |
| `.compare-block` | Composite class on the sticky summary + feature table sections; hidden on mobile. |
| `.fb-*` | Feedback modal: `.fb-card`, `.fb-survey-wrap`, `.fb-panel-left`, `.fb-right-col`, `.fb-smiley`, `.fb-chip`, `.fb-btn-primary`, `.fb-conf`, `.fb-gift`, `.fb-cta-badge`, `.fb-lock`. |
| `.sp-wrap` / `.sp-track` / `.sp-logo` | Social-proof marquee primitives. |
| `.illus-basic`, `.illus-pro`, `.illus-gold`, `.illus-enterprise` | Inline base64 PNG illustrations on the Individual cards. |
| `.usd` | Custom currency token — slightly smaller `$` glyph used before each price. |

### Naming convention
**Keep these names if you can** — designers refer to them. The `.pro-*` family is shared with the team Pro card; if you split bump UI to Gold (e.g. on `gold-highlight.html`), the JS still keys on `.pro-*` selectors — only the visual treatment moves.

---

## 14. Asset map

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

**Inline (base64) in v2.html:**
- `.illus-basic`, `.illus-pro`, `.illus-gold`, `.illus-enterprise` — small Individual-card illustrations (no extra HTTP requests).

**Sticky-footer logos** are loaded from Figma's MCP asset URLs (`https://www.figma.com/api/mcp/asset/…`). **These expire after 7 days — replace with permanent CDN URLs (or self-hosted assets) before shipping.**

External (CDN-only):
- Tailwind Play CDN
- Google Fonts: Inter (400–900), Hedvig Letters Serif, Instrument Serif, Roboto Condensed (400–700)
- Phosphor Icons (regular / bold / fill weights)
- *(optional / development only)* React 18 + ReactDOM 18 + agentation@3 from esm.sh

---

## 15. Production checklist — what to strip / replace

| Item | Action |
|------|--------|
| `<div id="agentation-root"></div>` + the `<script type="module">` block that mounts Agentation | **Remove entirely.** Design-review tool only. |
| `const DEV_RESET_ON_LOAD = true;` | **Flip to `false`** so the bump persists across reloads inside its 60-minute window. |
| Sticky-footer logo URLs (`https://www.figma.com/api/mcp/asset/...`) | **Replace** with permanent CDN / self-hosted URLs (Figma asset URLs expire after 7 days). |
| Tailwind Play CDN | Replace with your build's Tailwind / your own utility system. |
| Google Fonts CDN | Self-host or use your existing font pipeline. |
| Phosphor Icons CDN | Self-host or bundle. |
| Hard-coded prices ($9, $20, $99, $199, $5000 / 50000 credits, etc.) | Move to a config / pricing service. The labels are inline strings — grep and replace. |
| `localStorage` bump persistence | Move to **server-side** offer state, keyed by user. The 60-min window should start when the offer is granted by the backend, not when the page loads. |
| Empty `data-pro-seg` anchor on Pro | Optional — drop it once the period-handling code path is removed in your rewrite. |
| `body[data-top-view="team"]` CSS rule | Either keep + drive from the same body attribute in your framework, or fold into per-view conditional rendering. |
| Feedback modal copy + chip values | Localize. The smiley + chip selections post nothing in the prototype — wire to your analytics / survey backend. |

---

## 16. Open questions for product

1. **Bump granularity** — is the bump per-session (current), per-day, or once-ever per user? The prototype gates on `sessionStorage.proBumpOffered` (currently unused). Confirm the production semantics.
2. **Tablet (769–1279px)** — v2 has no `transform: scale()` fallback; the desktop layout horizontally scrolls. Confirm acceptable, or apply the scale pattern from `export-pricing.html`.
3. **`AI Credits (annual)` label** in the comparison table stays *"annual"* even when Gold is switched to monthly. By design we do **not** retitle the row on period switch — confirm this copy is intended.
4. **Custom seat input range** — clamped at 10–999 in the prototype. Confirm production max + whether seat counts above e.g. 50 should redirect to the Enterprise sales contact.
5. **Pro Monthly credits** — irrelevant for v2 (no monthly), but the baseline carries an `X,XXX` placeholder. If product ever opens monthly Pro, decide the credit number first.
6. **Submit gating on the survey** — currently both sentiment + improvement chip required. Confirm this is the intended bar for unlocking the bump.

---

## References

- **Full repo README** (covers all 7 prototype pages, not just v2): [README.md](https://github.com/manivasakan-arch/pricing-20-4-2026/blob/manivasakan-arch/dev-api-changes/README.md)
- **Live preview:** https://manivasakan-arch.github.io/pricing-20-4-2026/v2.html
- **.docx version of the engineering doc:** [Pricing-Page-Developer-Doc.docx](https://github.com/manivasakan-arch/pricing-20-4-2026/blob/manivasakan-arch/dev-api-changes/Pricing-Page-Developer-Doc.docx)

If anything in this doc drifts from what's actually in `v2.html`, the file is the source of truth — every behavior is implemented inline and easy to grep.
