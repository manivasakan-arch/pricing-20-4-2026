# pricing-20-4-2026

A static prototype repo containing **seven self-contained HTML pages** that together cover the current Presentations.ai pricing-and-discount flow plus a set of branching design explorations. There is no build step — every page is a single HTML file that pulls Tailwind, Inter, Phosphor icons, and (where relevant) React + Agentation from public CDNs.

> **Live preview (GitHub Pages):** [https://manivasakan-arch.github.io/pricing-20-4-2026/](https://manivasakan-arch.github.io/pricing-20-4-2026/)

---

## Table of contents

1. [Quick start](#quick-start)
2. [Page index](#page-index)
3. [Shared mechanics](#shared-mechanics)
4. [Page-by-page actions & flows](#page-by-page-actions--flows)
5. [Pricing model](#pricing-model)
6. [Asset map](#asset-map)
7. [Mobile responsive behavior](#mobile-responsive-behavior)
8. [Local development & deploy](#local-development--deploy)
9. [Known gaps / pre-ship checklist](#known-gaps--pre-ship-checklist)

---

## Quick start

```bash
# from the repo root
python3 -m http.server 8912
# then visit http://localhost:8912/index.html (or any other page below)
```

| Page | Local | Live |
|---|---|---|
| `index.html` | http://localhost:8912/ | https://manivasakan-arch.github.io/pricing-20-4-2026/ |
| `v2.html` | http://localhost:8912/v2.html | https://manivasakan-arch.github.io/pricing-20-4-2026/v2.html |
| `export-pricing.html` | http://localhost:8912/export-pricing.html | https://manivasakan-arch.github.io/pricing-20-4-2026/export-pricing.html |
| `gold-highlight.html` | http://localhost:8912/gold-highlight.html | https://manivasakan-arch.github.io/pricing-20-4-2026/gold-highlight.html |
| `dashboard-flash-sale.html` | http://localhost:8912/dashboard-flash-sale.html | https://manivasakan-arch.github.io/pricing-20-4-2026/dashboard-flash-sale.html |
| `dashboard-offer-expiry.html` | http://localhost:8912/dashboard-offer-expiry.html | https://manivasakan-arch.github.io/pricing-20-4-2026/dashboard-offer-expiry.html |
| `crazy8s.html` | http://localhost:8912/crazy8s.html | https://manivasakan-arch.github.io/pricing-20-4-2026/crazy8s.html |

---

## Page index

| # | File | Purpose |
|---|------|---------|
| 1 | `index.html` | Canonical pricing page — Individual / Teams. Pro card has a Monthly ↔ Annual toggle. |
| 2 | `v2.html` | Variant of `index.html` with the Pro billing-period control removed (annual-only). |
| 3 | `export-pricing.html` | Adds a one-time **Single Export** ₹1,999 card to the Individual view, separated by an "OR" divider. Includes an API tab + dev-console modal. |
| 4 | `gold-highlight.html` | Visual variant where **Gold** is the featured card (instead of Pro) on Individual + Team. |
| 5 | `dashboard-flash-sale.html` | Logged-in dashboard mock with the Pro discount chip and Upgrade flow in the top header. |
| 6 | `dashboard-offer-expiry.html` | 4-direction exploration of what happens after the 60-min discount expires. |
| 7 | `crazy8s.html` | Branching design iteration via the **crazy8s** workflow. Currently round 2 (8 directions) branched off the round 1 winner. |

---

## Shared mechanics

These behaviors appear across multiple pages.

### 1. Top segmented control (`setTopView`)
- Buttons: **Individual** / **Teams & Enterprise** / (sometimes **API**)
- `setTopView(target)` toggles the `.hidden` class on `#view-individual` / `#view-team` / `#view-api`, syncs `aria-selected`, swaps testimonial blocks (`[data-testimonials="individual"]` ↔ `[data-testimonials="team"]`), updates `body.dataset.topView`, and triggers `renderCompareSummary()` so the comparison table reacts.

### 2. Pro card price + bump (`renderPro`)
- Active button on the Pro card's `[data-pro-seg]` plan-segment determines the period.
  - **Annual** (default): `$20/mo billed annually`. With the bump active, `$15/mo` (with a `$20` strikethrough).
  - **Monthly**: `$40/mo billed monthly` + a green inline link "Switch to annual to save $240 per year".
- The **bump** is a 60-minute, $60-off, one-shot offer triggered when the user dismisses the close-intent feedback modal. State stored in `localStorage.proBumpStart=<timestamp>`. While active:
  - The Pro card price drops to $15/mo (annual).
  - A small orange `pro-discount-chip` reads `$60 off · ⏱ Ends in MM:SS` and bobs above the Buy Now button (vertical bob + halo pulse).
  - The sticky compare-summary mirrors the same chip via `pro-bumped-pill`.
- Credits in the Pro card list update with the period: **Annual = 5,000**, **Monthly = X,XXX** (placeholder, pending product input).
- `DEV_RESET_ON_LOAD = true` near the top of the script wipes bump + survey state on every load so the prototype always starts clean. Flip to `false` before shipping.

### 3. Gold card price (`render(period)` inside the Gold IIFE)
- `Annual = $99/mo billed annually` (default). `Monthly = $199/mo billed monthly`.
- Switching to monthly shows the green save-link "Switch to annual to save $1,200 per year".
- Credits in the Gold card list update with the period: **Annual = 50,000**, **Monthly = 5,000**.
- Exposed globally as `window.renderGoldIndividual` so `renderCompareSummary` can repaint the Gold card on view-change.

### 4. Team view shared seat-count (`teamState`)
- Pro Team and Gold Team cards share `teamState = { n, custom, hasTyped }` so the seat count on one card mirrors on the other.
- Default 5 seats. Chips: 3 / 5 (active, with `MOST BOUGHT` badge) / 10+ (becomes a `– [number] +` stepper input on click; clamped 10–999).
- **Pro tier rates:** 1–4 = $20/seat, 5–9 = $18/seat, 10+ = $17/seat. **5,000 credits per seat / yr.**
- **Gold tier rates:** 1–4 = $99/seat, 5–9 = $89/seat, 10+ = $84/seat. **50,000 credits per seat / yr.**
- The Buy CTA on each team card reads `Buy {n} Seats`.

### 5. Compare summary (`renderCompareSummary`)
The sticky mini-summary above the feature comparison table reacts to view + state:
- **Team view:** rewrites the Pro/Gold price columns to **`$<n × tierRate>`**, cadence reads `"N seats · billed annually"`, CTAs become **`Buy Pro N Seats`** / **`Buy Gold N Seats`**, and the credits row in the AI-Features table updates to **`n × creditsPerSeat`** for Pro and Gold.
- **Individual view:** Gold summary resets to the static `$99 / Buy Gold` defaults, the Pro CTA resets to `Buy Pro`, the Pro-credits cell stays at `5,000` and Gold-credits at `50,000`. `renderPro()` and `window.renderGoldIndividual(...)` are then called to repaint each card.
- A `body[data-top-view="team"]` CSS rule retargets every `.pricing-grid` from 4 columns to 3 and hides every 4n+2 child (the Basic slot).

### 6. Feedback modal — close-intent survey (`window.openFeedbackModal`)
- Triggered when the user clicks the outer ✕ Close button **and** they haven't already submitted the survey or claimed the bump in the current session.
- Two-step flow:
  1. **Survey** — pick a sentiment smiley + pick one improvement chip. Submit is disabled until both are chosen.
  2. **Success** — "Thank you. We've unlocked $60 off Pro Annual for the next one hour." Counts down via `setInterval`. Clicking **Buy Pro Annual at $60 off** (or **Claim**) calls `window.applyProBump()` which writes the bump timestamp, forces Pro to annual, repaints the Pro card with the discounted $15 price + bump chip, and starts the 60-min countdown.
- Session flags `proFeedbackSubmitted` / `proBumpOffered` prevent re-prompting in the same tab.

### 7. Developer Console modal (API tab)
- API-tab CTAs `[data-buy-now] data-plan="<tier>"` open the `#dc-modal` confirmation dialog.
- Modal copy: "Continue to the Presentations AI Developer Console — You're about to leave Presentations.ai…".
- **Continue** opens `https://console.presentations.ai` in a new tab. **Cancel** / **Esc** closes it.

### 8. Agentation toolbar
- All pages mount `agentation@3` via `esm.sh` ES-module imports for in-page design feedback (click any element, leave a note, copy a structured payload).
- **Remove these import + mount blocks before shipping to production.**

---

## Page-by-page actions & flows

### 1. `index.html` — Canonical pricing page
**Audience views:** Individual + Teams & Enterprise (the API tab was removed from this page; see `export-pricing.html` for the API view).

**Visual frame**
- Outer modal-style container at 1280px wide, white card on a light page.
- Title "Pick a plan that grows with you", a top-right ✕ Close button, and a thin segmented control (Individual / Teams).

**Individual view** (`#view-individual`)
- Three cards: **Basic** (left, $9/mo annual), **Pro** (center, featured — `pro-card-bg`, full borders, 2px brand-secondary edge, floats up 21px, taller card), **Gold** (right, $99/mo annual).
- Pro card has a Monthly ↔ Annual seg-control + price block + Buy Now CTA (with the bump-state chip floating above when active) + features list.
- Gold card has its own Monthly ↔ Annual seg-control and the green "Switch to annual to save $1,200 per year" link in monthly mode.
- Hovering any card surfaces a `.plan-hint` tooltip above it ("For those who want to make simple decks occasionally" etc.). Hidden on mobile.

**Team view** (`#view-team`)
- Three cards: **Pro Team** (featured), **Gold Team**, **Enterprise**.
- Pro Team and Gold Team share the seat-count chip row (3 / 5 / 10+); changing one updates the other and triggers `renderAllTeams()` → repaints both cards + the compare summary.
- Enterprise card carries a "Contact Sales" CTA (no pricing, custom subtitle "tailored for your organization").

**Below the cards** (`#below-views`)
- 800px spacer reserves vertical space for the absolutely-positioned card row.
- Testimonials (3 cards): `data-testimonials="individual"` (Patrick / Angela / Walter) shown on Individual; `data-testimonials="team"` (Marcus / Desmond / Elena, hand-drawn portraits in `assets/testimonials/`) shown on Team.
- Sticky **MINI PLAN SUMMARY** that mirrors the cards' prices/CTAs and pins to the top of the viewport on scroll.
- **FEATURE COMPARISON TABLE** with sections (AI / Sharing / Collaboration / Brand / Analytics / Integrations). Tooltip ⓘ icons on each row use Phosphor `ph ph-info`.
- **Trust strip** with AICPA SOC 2 + GDPR badges and the caption "We're a SOC 2 Type II, GDPR-compliant organization."
- 60px bottom spacer before the sticky social-proof footer (logo marquee).

**Sticky social-proof footer**
- "❤️ Loved by 10M+ presenters at" + a continuous left→right logo marquee (Microsoft, Google, Adobe, Meta, McKinsey, Amazon, Notion, EY, BCG, …). The track is duplicated for seamless loop. Hover pauses the animation.

**Close-intent flow**
- The outer ✕ button intercepts the click and opens the feedback modal (unless the survey was already submitted or the bump is active). See *Shared mechanics §6*.

---

### 2. `v2.html` — Pro annual-only variant
- Identical to `index.html` **except** the Pro card has no billing-period seg-control. The `<div class="plan-seg" data-pro-seg>` element is kept as a **hidden anchor** (empty `aria-hidden="true"` div) so that `renderPro` / `getProPeriod` selectors still resolve — `getProPeriod()` falls through to its `'annual'` default and the price stays locked at $20/mo annual.
- Use this when running pricing experiments where you don't want to expose monthly billing for Pro.

---

### 3. `export-pricing.html` — Individual + Single Export + API
Adds a one-time PowerPoint export option to the Individual view, plus the full API tab.

**Individual view**
- **Single Export card** (leftmost, `id="card-single-export"`): ₹1,999 one-time purchase, "Get a single PPT" CTA, "For one time usage" caption.
- An **OR divider** (`#or-divider`) — vertical dashed line with a centered "OR" label — separates Single Export from the subscription trio (Basic / Pro / Gold).
- Card row is widened to **1488px** (Single + OR + Basic + Pro + Gold). The page applies a CSS scale so it never touches the viewport edge:
  ```css
  body > .relative.mx-auto {
    transform: scale(min(1, calc((100vw - 160px) / 1488px)));
    transform-origin: top center;
  }
  ```
  Above ~1648px viewport this is 1× (no scale); below it scales down proportionally.

**Team view** — same as `index.html`.

**API view** (`#view-api`)
- Three usage tiers: **Starter $199/mo**, **Growth $499/mo (featured)**, **Scale $1,999/mo**.
- All CTAs say **"Explore on Developer Console"**. Clicking opens the `#dc-modal` confirmation modal (see *Shared mechanics §7*).
- **#api-extras** section appears below the cards (only when API view is active):
  - **Method** — two integration cards (REST API + MCP Integration).
  - **End points** — three endpoint cards (`/Topic`, `/File`, `/SingleSlide`) with preview-image footers in `assets/api/`.

**Mobile (≤768px) extras specific to this page**
- Single Export card pinned **first**, followed by a **horizontal** OR divider (the vertical dashed line is restyled to a horizontal rule with the "OR" label sitting on top), then Pro / Basic / Gold.

---

### 4. `gold-highlight.html` — Gold-as-featured variant
Same structure as `index.html` but the **featured-card treatment is moved from Pro → Gold** on both the Individual and Team views.

**Differences**
- Pro card becomes plain (`bg-white border border-primary border-l-0`, no float-up, regular height, outline button).
- Gold card gets `pro-card-bg border-2 border-brand-secondary rounded-[4px] shadow-card`, floats up 21px, gains the `pt-[45px]` inner padding, and uses the filled `pro-btn` CTA.
- Same swap on the Team tab: Pro Team becomes plain (with left-rounded corners since it's leftmost), Gold Team is featured at center.
- The "Export to PowerPoint and Google Slides" bullet is removed from Basic, Pro Individual, and Pro Team (Gold's content is unchanged).
- Card heights enlarged so the **highlighted Gold card extends 120px below its last bullet**, and plain cards bottom-align with it (Gold = 720px height, plain = 699px height).

Use this when validating whether shifting feature-emphasis to Gold improves upgrade conversion vs the canonical Pro highlight.

---

### 5. `dashboard-flash-sale.html` — Logged-in dashboard with discount chip
A full-width dashboard mock simulating the in-product top header for a Pro upgrade flash discount.

**Layout**
- **Sidebar (260px):** workspace switcher (avatar + "Nvidia Works…" + inline orange Upgrade link + caret), nav (Home active / Created by me), Projects (`+ Create project` with `PRO` badge), Resources (Hire an expert / Templates), footer (Workspace settings / Invite new members `PRO`).
- **Top nav (right-aligned):**
  - **`$60 off · 🕒 Ends in MM:SS`** chip — verbatim copy of `.pro-bumped-pill` from `index.html`. Bob + glow keyframes, right-side wedge (8×8 rotated square at z-index `-1` so it tucks behind the chip body) aimed at the Upgrade button.
  - **Trial / Credits pill** — white background with a 12% brand-orange stroke. Inside: a rotating phrase (`"🔥 Trial: 4 days left"` ↔ `"✨ Credits: 60 left"` — vertical-loop animation via a 3-line track with the first line duplicated at the end for seamless wrap) + an orange filled **Upgrade** button with a diagonal **shimmer sweep** every 2.6s.
  - Three utility icon buttons (search / help / bell) and a 28×28 avatar.

**Below the header**
- Greeting: "Great work on your last deck, John. What will you create next?"
- Three quick-action cards (`Paste an outline`, `Upload a file or share a link`, `Start with a prompt`) using the supplied artwork in `assets/dashboard/qa-*.jpg`. Capped at 960px max-width so the buttons stay tight.
- **Recent** grid: three cards with **16:9 thumbnails** using `assets/dashboard/recent-*.png`.

**Discount countdown**
- Persists across reloads via `localStorage.upDiscountEndsAt`. 60-minute fresh window each time it hits zero.
- Shared with the modal timer (when applicable).

---

### 6. `dashboard-offer-expiry.html` — Post-expiry exploration
A 2×2 gallery comparing four directions for *what happens once the discount countdown reaches zero*. Each card is wrapped in a mini top-nav so reviewers see the treatment in context, and has its own **`Trigger expiry`** + **`Re-arm offer`** controls plus a fast 12-second demo countdown.

| # | Approach | Behavior on expiry |
|---|----------|--------------------|
| **A** | Inline gray swap | Same chip slot turns gray with **"Offer expired · Try again"** copy. Clicking the gray pill itself re-arms the offer. |
| **B** | Slide-in toast (recommended default) | Chip dims; a 360px toast slides into the top-right with **Generate again** + **No thanks** + close ✕. |
| **C** | Modal lockout | Page dims with a scrim and a centered modal: *"You missed the $60 off offer."* with **Generate one more chance** + **No thanks**. |
| **D** | Auto cooldown | Chip flips to a *"Charging next offer…"* progress pill. The orange progress bar fills and the offer auto-rearms after the cooldown (6s for the demo). |

The footer of the page contains a suggested rollout note (default to B, cap re-arms at 2/session, watch click-through on "Generate again").

---

### 7. `crazy8s.html` — Branching design iteration (current: Round 2)
Driven by the **crazy8s** skill — one rolling file, 8 directions per round, the user picks one, the next round pins that choice as variant 01 and produces 7 fresh branches off it.

**Round 1 (history, no longer in the file)** — 9 directions for the post-expiry recovery: inline gray swap, slide-in toast, modal lockout, auto cooldown, sticky page banner, bell badge + flyout, stepped FOMO ($30 last chance), vacuum into Upgrade, **regen chip + hover tooltip**.

**Round 2 (current)** — branched off **R1·09** (regen chip + hover tooltip), varying *how the reactivation surface is presented in the header*. Constants: brand orange `#ff5500`, the dashboard top-nav frame, the 60-min revival behavior.

| # | Direction | Behavior on expiry |
|---|-----------|--------------------|
| **01** *(pinned from R1)* | Regen chip + hover tooltip | Chip becomes **"↻ You missed the $60 off Pro Annual offer • Reactivate"**. Hover shows a dark tooltip: *"You've got one reactivation left — Click to claim it. This is the last time we can bring it back."* Click reactivates. |
| **02** | Ghost chip + caption | Outlined orange **`Reactivate`** pill next to a muted always-readable caption *"$60 off Pro Annual · last chance"*. No hover required. |
| **03** | Sub-nav strip | Chip vanishes; a thin orange-tinted strip drops below the top-nav with the message + Reactivate button + dismiss ✕. |
| **04** | Nested in the trial pill | Chip slot empties; the existing trial pill grows a "↻ Reactivate $60 off" segment between the trial copy and the Upgrade button. |
| **05** | Icon badge + popover | Chip shrinks to a small orange icon button with a "1" badge. Click opens a popover with the message + Reactivate / Maybe later. |
| **06** | Slide-in invitation card | Chip vanishes; a small branded card slides in from the right of the top-nav with full reactivation messaging + Reactivate CTA + close ✕. |
| **07** | Marquee ticker | Thin orange marquee strip scrolls above the top-nav with looping copy. Click anywhere on the strip to reactivate. |
| **08** | Split pill | Single segmented pill — gray *"Offer expired"* segment on the left, orange *"Reactivate"* segment on the right. |

Each variant card has its own scoped `Trigger expiry` / `Re-arm offer` controls and runs a fast 12-second demo countdown so reviewers can flip the state instantly.

---

## Pricing model

Numbers used across the prototype:

| Plan | Annual (per user) | Monthly | Credits |
|------|-------------------|---------|---------|
| Basic | $9/mo billed annually | — | 1,500 / yr |
| Pro | **$20/mo billed annually** ($15 with bump) | $40/mo | 5,000 (annual) / X,XXX (monthly placeholder) |
| Gold | $99/mo billed annually | $199/mo | 50,000 (annual) / 5,000 (monthly) |

Team tiers (per seat / yr):

| Plan | 1–4 | 5–9 | 10+ | Credits / seat / yr |
|------|-----|-----|-----|---------------------|
| Pro Team | $20 | $18 | $17 | 5,000 |
| Gold Team | $99 | $89 | $84 | 50,000 |

API tiers (`export-pricing.html`):

| Plan | Price | Slides / mo | Credits / mo |
|------|-------|-------------|--------------|
| Starter | $199/mo | 400 | 2,000 |
| Growth (featured) | $499/mo | 1,200 | 6,000 |
| Scale | $1,999/mo | 6,000 | 30,000 |

**Bump:** $60 off Pro Annual, 60-minute window. Stored in `localStorage.proBumpStart`. `DEV_RESET_ON_LOAD = true` resets the state on every load while iterating.

**Single Export** (`export-pricing.html` only): ₹1,999 one-time PPT.

---

## Asset map

```
belo-horizonte/
├── index.html                 # canonical pricing
├── v2.html                    # Pro annual-only variant
├── export-pricing.html        # Single Export + API tab
├── gold-highlight.html        # Gold-as-featured variant
├── dashboard-flash-sale.html  # logged-in dashboard with discount chip
├── dashboard-offer-expiry.html  # 4-direction expiry exploration
├── crazy8s.html               # branching iteration (current round 2)
├── orange.html | amber-palette.html | chip-colors.html | seats-10plus.html | seats-10plus-chip.html
│   # legacy palette + seat-input explorations
├── package.json | package-lock.json
│   # only used to host the agentation dev dependency
├── Pricing-Page-Developer-Doc.docx
│   # full engineering hand-off doc generated earlier
└── assets/
    ├── api/                   # API tier card art + endpoint preview tiles
    │   ├── starter.png growth.png scale.png
    │   └── ep-topic.png ep-file.png ep-singleslide.png
    ├── dashboard/             # quick-action icons + Recent thumbnails
    │   ├── qa-outline.jpg qa-upload.jpg qa-prompt.jpg
    │   └── recent-mckinsey.png recent-aethelgard.png recent-anatomy.png
    ├── export/                # Single Export PowerPoint icon
    │   └── single-export.png
    ├── testimonials/          # team-tab portraits
    │   └── team-marcus.png team-desmond.png team-elena.png
    └── trust/                 # SOC 2 + GDPR badges
        └── soc2.svg gdpr.svg
```

External (CDN-only): Tailwind Play CDN, Google Fonts (Inter + Hedvig Letters Serif + Instrument Serif + Roboto Condensed), Phosphor Icons (regular / bold / fill), React 18 + ReactDOM 18 + agentation@3 from esm.sh.

The sticky-footer logo marquee (`Microsoft, Google, Adobe, Meta, McKinsey, Amazon, Notion, EY, BCG`) is **served from Figma's MCP asset URLs which expire after 7 days** — replace with permanent CDN URLs (or inline) before shipping.

---

## Mobile responsive behavior

The viewport meta is `width=device-width, initial-scale=1` on all pages so media queries fire on real phones. A single `@media (max-width: 768px)` block in each pricing page rewires the layout:

- **Outer stage:** full width, no scaling, padding `0 16px 96px` (96px bottom reserves room for the sticky social-proof footer).
- **Top segmented control:** sticky to `top: 0` with `z-index: 50`, white background + bottom border.
- **Outer ✕ Close button:** `position: fixed; top: 10px; right: 10px; z-index: 60` so it stays visible while scrolling.
- **Each `#view-*` container:** `display: flex; flex-direction: column; gap: 12px`. `.hidden` is honored so only the active view's cards render.
- **Cards:** `position: static; width: 100%; max-width: 420px`, centered, with consistent `border-radius: 4px`. **`.pro-card-bg` (the featured card) gets `order: -1`** so the Pro card always lands at the top of the stack on mobile — including on `gold-highlight.html` where Gold carries `pro-card-bg`.
- **Hover tooltips (`.plan-hint`)** are hidden — they don't make sense on touch.
- **Comparison table block (`.compare-block`)** is hidden — it doesn't scale gracefully on phones.
- **Site footer:** stays `position: fixed; bottom: 0` with a tighter caption + the full-width logo marquee underneath.
- **Feedback modal:** stacks vertically with the orange offer panel (gift / tag / `$60 off Pro Annual` title) on top and the survey form below.

`export-pricing.html` mobile additions:
- Single Export card pinned first (`order: -3`), then a **horizontal** OR divider (`order: -2`), then Pro / Basic / Gold.
- Seat-chip-row's fixed 312px width released to 100% so chips line up edge-to-edge with the Buy-N-Seats button.

`dashboard-flash-sale.html` is desktop-only by design — no mobile breakpoints (per the design brief).

---

## Local development & deploy

### Local
```bash
python3 -m http.server 8912
# open http://localhost:8912/<page>.html
```
No build step. Edits to `.html` are picked up on refresh.

### Public preview (GitHub Pages)
The fork at `github.com/manivasakan-arch/pricing-20-4-2026` has Pages enabled on the `manivasakan-arch/dev-api-changes` branch at `/`. Pushing to that branch rebuilds in ~30 seconds.

```bash
git push fork manivasakan-arch/dev-api-changes
```

Live root: https://manivasakan-arch.github.io/pricing-20-4-2026/

### Adding more pages
Drop a new HTML file at the repo root and `git push` — it'll be served at `https://manivasakan-arch.github.io/pricing-20-4-2026/<file>.html` after the next Pages build.

---

## Known gaps / pre-ship checklist

- **Set `DEV_RESET_ON_LOAD = false`** so the bump persists across refreshes inside its 60-minute window.
- **Replace the sticky-footer Figma-hosted logo URLs** — they 404 after 7 days.
- **Decide the real Pro-monthly credit number** — currently a literal `X,XXX` placeholder.
- **Remove the Agentation `<div id="agentation-root"> + <script type="module">` block** for the production build (keep it for design review).
- Pricing labels are hard-coded inline. If a CMS or A/B framework will drive them, extract to a config map.
- Tablet viewports (769–1279px) on `index.html` / `v2.html` / `gold-highlight.html` rely on browser auto-scale; only `export-pricing.html` ships an explicit `transform: scale()` fallback. Add the same fallback to the others if tablets matter.
- `"AI Credits (annual)"` label in the comparison table stays *"annual"* even when a card is switched to monthly. By design we don't refactor the table on period switch — confirm this copy is intended.
- The team-Pro card carries the `pro-card-bg` class and currently powers the bump-state chip lookup. If you split the bump UI to Gold (e.g. on `gold-highlight.html`), update `pro-cta-wrap`, `pro-discount-chip`, `pro-bumped-pill` selectors accordingly — only the visual treatment was swapped, the JS is still keyed on the Pro selectors.

---

## Conventions

- **No build tooling** — every page is self-contained HTML with inline `<style>` + `<script>` and CDN imports.
- **Tailwind utility classes** for layout + spacing; hand-rolled CSS classes (`.pro-card-bg`, `.plan-seg-btn`, `.seat-chip-row`, `.feature-row-label`, `.compare-block`, `.up-discount-chip`, `.regen-chip`, etc.) carry component-specific visual treatments.
- **Tokens** declared in the inline `tailwind.config`: `brand.500 = #ff5500`, `ink.{primary, secondary, tertiary}`, `success.fg = #16a34a`, `border.{primary, secondary}`.
- **Cross-IIFE coordination** through `window` globals: `window.openFeedbackModal`, `window.applyProBump`, `window.renderGoldIndividual`.
- **Per-page `<title>` reads "<feature> — <variant>"** so tabs stay legible during multi-page review.

---

If anything in this README drifts from what's actually in the file, the source of truth is the page itself — every behavior is implemented inline and easy to grep.
