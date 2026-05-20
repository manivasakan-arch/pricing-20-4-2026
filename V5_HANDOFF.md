# v5 — Pricing close-nudge handoff

> Engineering reference for the `before-you-go` discount popup that
> replaces the `share-feedback` testimonial nudge on the v5 pricing
> page. Use this together with the source file at `v5.html`.

- **Live**: https://manivasakan-arch.github.io/pricing-20-4-2026/v5.html
- **Local**: http://localhost:8912/v5.html
- **Source**: `belo-horizonte/v5.html` (single self-contained HTML — no
  bundler, no build step; Tailwind Play CDN + inline `<style>` +
  inline `<script>`)
- **Figma**: AMJ-26 Growth · node 217:7704
  https://www.figma.com/design/iuz5SLIPsMDKLgEPdBmq6W/AMJ-26---Growth?node-id=217-7704

---

## 1. What the popup does

When the user clicks the **X (Close)** on the pricing modal in the
header chrome (`#close-pricing-btn` on the host page) the discount
popup intercepts the dismiss and slides in over a black scrim. It
animates a `$60 off` reveal, enables the **Buy Pro Annual at $60 Off**
CTA after the reveal finishes, and — on dismiss — drops the user
back to the pricing page with the Pro Annual bump applied.

The popup is *not* a feedback collector: there is no submission, no
form, no testimonial. It is purely a save-the-conversion surface.

```
Pricing page close (X)
    │
    ▼
v5 discount popup opens     ── (Esc / X both dismiss)
    │   ┌──────────────────────────────────┐
    │   │  Dial spin → CTA reveal → pill   │
    │   │  Auto-cycle features every 4s    │
    │   │  User can click features to swap │
    │   └──────────────────────────────────┘
    │
    ▼
Popup close
    │
    ├─ sessionStorage.proOfferSeen = 'true'
    └─ window.applyProBump()
              │
              ▼
         Pro card on the page
         shows blue chip + bumped price
```

---

## 2. Trigger wiring (host page → popup)

The host page exposes a global `window.openDiscountPopup()` and the
pricing modal's close handler calls it instead of the legacy
`window.openFeedbackModal()` it used in v2/v3.

```js
closeBtn.addEventListener('click', (e) => {
  if (sessionStorage.getItem('proFeedbackSubmitted') === 'true') return;
  if (bumpRemaining() > 0) return;
  if (typeof window.openDiscountPopup !== 'function') return;
  e.preventDefault();
  e.stopPropagation();
  if ((document.body.dataset.topView || 'individual') !== 'individual'
      && typeof setTopView === 'function') {
    setTopView('individual');
  }
  window.openDiscountPopup();
});
```

Skip conditions (intentional):

- `sessionStorage.proFeedbackSubmitted === 'true'` — user already
  completed a feedback flow earlier in the session.
- `bumpRemaining() > 0` — bump is already running; no need to nudge.
- Team tab → forced back to `individual` first so the underlying
  Pro card reflects the bump.

---

## 3. DOM structure

```html
<div class="dp-backdrop" id="dp-backdrop" role="dialog" aria-modal="true"
     aria-labelledby="dp-box-title" data-open="false">
  <div class="dp-stack">
    <!-- Top eyebrow strip — slides up out from behind box top on open -->
    <div class="dp-eyebrow-strip">
      <span class="gift-icon" aria-hidden="true">🎁</span>
      <p>Before you go, a special discount, just for you for the next hour.</p>
    </div>

    <!-- Main box (720 × 468) -->
    <div class="dp-box">
      <h2 class="dp-box-title" id="dp-box-title" data-price-loading="true">
        Get
        <span class="accent">
          <span class="dp-d-sym">$</span>
          <span class="dp-digits">
            <span class="dp-digit" data-target="6"><span class="dp-digit-track"></span></span>
            <span class="dp-digit" data-target="0"><span class="dp-digit-track"></span></span>
          </span>
          <span class="dp-d-off">off</span>
        </span>
        Pro Annual and access premium features
      </h2>

      <!-- Left clickable feature list (6 items, each with data-preview + data-icon-active/inactive) -->
      <div class="dp-feat-list" id="dp-feat-list">
        <button class="dp-feat-item is-active" data-feat="export"
                data-preview="assets/v5/preview-v2.png"
                data-icon-active="ph-fill ph-export"
                data-icon-inactive="ph ph-export">…</button>
        …5 more items…
      </div>

      <!-- Right preview image — swaps on feature click -->
      <div class="dp-preview" aria-hidden="true">
        <img class="dp-preview-img" id="dp-preview-img"
             src="assets/v5/preview-v2.png" alt="" />
      </div>

      <!-- Timer pill above CTA, arrow points down to the button -->
      <div class="dp-timer-wrap">
        <span class="dp-timer-pill">
          <i class="ph-fill ph-clock"></i>
          <span>Ends in&nbsp;&nbsp;<span id="dp-timer">59:56</span></span>
        </span>
      </div>

      <!-- CTA — starts disabled, text upgrades after reveal -->
      <div class="dp-cta-wrap">
        <button class="dp-cta" id="dp-cta" type="button">Buy Pro Annual</button>
      </div>
    </div>

    <!-- Close (sibling of .dp-box so it's not clipped by overflow:hidden) -->
    <button class="dp-close" id="dp-close" type="button" aria-label="Close">
      <i class="ph ph-x"></i>
    </button>
  </div>
</div>
```

### Geometry

| Element | Size | Position |
|---|---|---|
| `.dp-eyebrow-strip` | 700 × 36 | sits above `.dp-box`; `border-radius: 4px 4px 0 0` |
| `.dp-box` | 720 × 468 | white; `overflow: hidden`; `z-index: 1` (covers strip during entry) |
| `.dp-box-title` | full width | absolute, `top: 30px`, `text-align: center`, 20px Inter Medium |
| `.dp-feat-list` | 230 wide | absolute, `top: 84px`, `left: 30px`, 10px gap between items |
| `.dp-preview` | 400 × 280 | absolute, `top: 84px`, `left: 290px`, `border: 1px solid rgba(10,25,37,0.09)` |
| `.dp-timer-wrap` | 152 wide | absolute, `top: 359px`, centered over CTA at `left: 490px` with `translateX(-50%)` |
| `.dp-cta-wrap` | 400 × 44 | absolute, `top: 394px`, `left: 290px` |
| `.dp-close` | 24 × 24 | absolute on `.dp-stack`, `top: 4px right: -36px` (outside box top-right) |

---

## 4. Entry animation sequence (open)

Times measured from `openDiscountPopup()` invocation.

| t (ms) | Step | Class / property |
|---:|---|---|
| 0 | `.dp-backdrop[data-open="true"]` toggled | opacity 0 → 1, visibility hidden → visible (200ms ease) |
| 0 | `.dp-stack` scales in | `translateY(8px) scale(0.98)` → `translateY(0) scale(1)` (250ms cubic-bezier(0.34, 1.35, 0.64, 1)) |
| 200 | `.dp-eyebrow-strip` reveals | `translateY(40px)` → `translateY(0)` (850ms cubic-bezier(0.16, 1, 0.3, 1)) — emerges from behind box top |
| 0 | Dial spawned, blurred, parked at index 0 | `data-price-loading="true"` applies `filter: blur(6px)` to `.dp-d-sym` + `.dp-digit-track` |
| 700 | Dial released | tracks `translateY(-(cycles * 10 + target) * 24px)` over 3.2s / 3.5s (per-digit) cubic-bezier(0.33, 1, 0.68, 1); blur fades in parallel |
| 3600 | CTA + pill reveal | `ctaBtn.disabled = false`, text → `"Buy Pro Annual at $60 Off"`, class `is-revealed` added (starts shimmer sweep); `.dp-timer-pill.is-hidden` removed |
| ~4200 | Auto-cycle begins | `startAutoCycle()` advances feature items every 4s; cancels on user click |

Reduced-motion fallback: digit dial relies on CSS `transition`; if the
client honours `prefers-reduced-motion`, transitions still fire (we
don't gate on the media query) but the cycle interval keeps cadence —
review with design before launch if motion-reduction is a hard
requirement.

---

## 5. State

### Per-popup lifecycle (in-memory)

| Variable | Purpose |
|---|---|
| `startMs` | popup-open timestamp; drives the visible `Ends in MM:SS` countdown |
| `interval` | 1s `setInterval` for the timer tick |
| `dialTimer` | 700ms timeout that flips the dial from blurred-park to spinning-to-target |
| `revealTimer` | 3600ms timeout that enables the CTA + reveals the pill + starts shimmer |
| `cycleInterval` | 4s rolling timeout that advances the active feature item |
| `userTookOver` | `true` once the user clicks any feature item — kills the cycle |

All five timers are torn down inside `close()` so re-opening replays
the sequence cleanly.

### Cross-page persistence

| Key | Storage | Set by | Read by |
|---|---|---|---|
| `proOfferSeen` | `sessionStorage` | `close()` (popup dismiss) | `renderPro()` on the host page — gates the blue chip above Buy Now |
| `proBumpStart` | `localStorage` | `close()` calls `window.applyProBump()` which writes the bump timestamp | `bumpRemaining()` / `renderPro()` |

`renderPro()` is called inside `close()` so the underlying Pro card
immediately reflects the bumped state (price $20 → $15, cadence $240
→ $180, blue `$60 off · Ends in MM:SS` chip surfaces above CTA).

---

## 6. Interactions

### Feature list

Each `.dp-feat-item` is a `<button>` carrying:

- `data-feat` — slug, e.g. `export`, `ai`, `analytics`
- `data-preview` — image path swapped into `#dp-preview-img`
- `data-icon-active` / `data-icon-inactive` — Phosphor icon class
  switched on activate/deactivate (e.g. `ph-fill ph-sparkle` ↔
  `ph ph-sparkle`)

`activateFeat(item)`:

1. Removes `.is-active` from every item, restores inactive icon class
2. Adds `.is-active` to the clicked item, swaps to fill icon
3. Applies preview swap if `src` differs

Preview swap is a two-phase blur-slide:

```css
.dp-preview-img.is-swapping  { opacity: 0; transform: translateX(-14px); filter: blur(6px); }
.dp-preview-img.is-entering  { opacity: 0; transform: translateX( 14px); filter: blur(6px); transition: none; }
.dp-preview-img              { transition: opacity 0.28s ease, transform 0.42s cubic-bezier(0.22, 1, 0.36, 1), filter 0.28s ease; }
```

JS: `is-swapping` → wait 240ms → swap `src` → `is-entering` → next
frame, drop `is-entering` so the transition runs from the
right-and-blurred starting position.

### Auto-cycle

Starts ~4.2s after `open()` (so it kicks in just after the CTA reveal
lands). Advances `(activeIdx + 1) % items.length` every 4s. User click
sets `userTookOver = true` permanently and clears `cycleInterval`.

### Close paths

All three dismiss paths route through `close()`:

- `dp-close` button click
- `Escape` key (only when `[data-open="true"]`)
- *(backdrop click intentionally does **not** close — design ask)*

`close()` writes `proOfferSeen` + calls `applyProBump()` regardless of
how the popup was dismissed. The intent is: any time the user saw the
reveal, they've earned the discount.

### CTA click

`dp-cta` redirects to `checkout.html?promo=PROANNUAL60`. The
`checkout.html` file does not exist in this repo — wire it to the real
checkout in production. The promo code is a placeholder.

---

## 7. Pricing math reference

The host page bakes the cadence text on initial render and re-renders
inside `renderPro()` / `setGoldPeriod()`:

| Card | Display | Cadence text |
|---|---|---|
| Basic | $9/mo (static) | `108 billed annually (incl. tax)` |
| Pro Annual (normal) | $20/mo | `240 billed annually (incl. tax)` |
| Pro Annual (bumped) | $15/mo (strikethrough $20) | `180 billed annually (incl. tax)` |
| Pro Monthly | $40/mo | `40 billed monthly (incl. tax)` |
| Gold Annual | $100/mo | `1200 billed annually (incl. tax)` |
| Gold Monthly | $200/mo | `200 billed monthly (incl. tax)` |
| Team Pro/Gold | $rate/mo × n seats | `${n} seats · ${n * rate * 12} billed annually (incl. tax)` |

The formula in the Pro card cadence is `${parseInt(annualNow, 10) * 12}`
where `annualNow` is `'15'` when bumped, `'20'` otherwise.

---

## 8. Custom properties / theme

| Token | Value | Where |
|---|---|---|
| `--shadow/drop-2` | `rgba(10,25,37,0.09)` | box shadow stops on `.dp-box`, `.dp-tile`, `.dp-cta` |
| `--shadow/inner-1` | `rgba(255,255,255,0.8)` | inset highlight on box + tiles |
| Primary blue | `#005eff` / `#0055ED` | `.dp-timer-pill`, `.dp-d-sym`, `.dp-digit-track > span`, `.dp-box-title .accent`, `.pro-discount-chip` |
| Hover-dark | `#23303B` | `.dp-cta:hover`, `.pro-btn:hover`, segmented controls |
| Box bg | `#ffffff` | `.dp-box` |
| Backdrop | `rgba(10,25,37,0.72)` + `backdrop-filter: blur(1px)` | `.dp-backdrop` |

---

## 9. JS API

| Function | Where defined | What it does |
|---|---|---|
| `window.openDiscountPopup()` | inline IIFE in `v5.html` | toggles `[data-open="true"]`, starts dial + timer + auto-cycle |
| `window.closeDiscountPopup()` | same | dismiss path; sets `proOfferSeen` + calls `applyProBump()` |
| `window.applyProBump()` | pricing-page render block | writes `localStorage.proBumpStart`, calls `renderPro()` |
| `bumpRemaining()` | pricing-page render block | minutes left on the bump (0 = expired) |
| `renderPro()` | pricing-page render block | repaints the Pro card (price, cadence, chip, hint) |

---

## 10. Asset inventory

```
assets/v5/
├── preview-v2.png            # Export preset (default active)
├── preview-ai-models.png     # AI models
├── preview-analytics.png     # Analytics
├── preview-shared-knowledge.png
├── preview-invite-team.png
├── preview-projects.png
├── gifts_app.svg             # multi-color gift SVG (currently unused — eyebrow uses 🎁 emoji)
├── dots-bg.png               # dotted backdrop (legacy from glassmorphic preview — kept but unused)
├── icon-ppt.png              # Microsoft Powerpoint tile icon
├── icon-gslides.png          # Google slides tile icon
├── icon-pdf.png              # PDF tile icon
└── icon-image.png            # Image tile icon
```

All previews are 800 × 544 PNGs sourced from Figma. They render at
400 × 280 inside `.dp-preview` with `object-fit: cover`.

---

## 11. Accessibility checklist

- `[role="dialog"]` + `aria-modal="true"` + `aria-labelledby="dp-box-title"`
- Esc closes the popup; backdrop click intentionally does not (design)
- `dp-close` is a `<button>` with `aria-label="Close"`
- All feature items are `<button>` elements (keyboard navigable; Enter / Space invoke `activateFeat`)
- All decorative `<img>` and `<i>` elements carry `aria-hidden="true"`
- Live timer (`#dp-timer`) updates the visible text; no `aria-live` (it's a soft signal, not status — review with a11y if needed)

---

## 12. Production checklist

Before shipping:

1. Wire `dp-cta` `href` from `checkout.html?promo=PROANNUAL60` to the
   real Stripe / billing checkout URL with the same promo code.
2. Server-enforce the **one reactivation per user** rule
   (currently `localStorage.proBumpStart` is client-only — anyone
   clearing storage can re-trigger).
3. Replace the placeholder PROANNUAL60 promo with the real coupon code.
4. Confirm Phosphor Icons CDN is allowed (or self-host the icon font).
5. Decide whether the backdrop-click should also dismiss (currently it
   doesn't — confirm with design before launch).
6. Confirm reduced-motion behaviour with accessibility — the dial
   spin and shimmer still play under `prefers-reduced-motion: reduce`.
7. Translate the eyebrow strip + title copy if launching outside en-US.
8. Replace `console.log` / unused legacy `fb-` markup (the
   testimonial picker DOM is kept hidden with `display: none !important`
   for back-compat; remove once v5 ships).

---

## 13. Files touched

| File | Role |
|---|---|
| `v5.html` | popup markup + CSS + JS, plus the underlying pricing page |
| `assets/v5/*.png` | preview tiles (one per feature) |
| `assets/v5/gifts_app.svg` | optional brand gift icon (not currently used) |

No external dependencies beyond Tailwind Play CDN, Phosphor Icons CDN,
and Google Fonts (Inter + Instrument Serif + Merriweather +
Roboto Condensed + Hedvig Letters Serif).
