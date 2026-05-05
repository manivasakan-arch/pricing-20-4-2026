# Dashboard Flash-Sale (Expired) — Engineering Handoff

**File:** `dashboard-flash-sale-expired.html` (~660 lines, single self-contained HTML)

**Purpose:** Top-nav demo of three header states for the post-trial discount flow:
- **Active** — discount countdown still running
- **Expired** — countdown elapsed, "missed offer" CTA prompts re-activation
- **Nudge** *("close nudge")* — soft "share feedback, get $60" prompt with underline-button claim and full Trial/Credits + Upgrade CTA

Bottom-right floating toggle switches between states. State swap fires `state-fade-in` animation + auto-routes back to Active when user clicks Reactivate / Upgrade / Claim Discount.

---

## 1. Tech stack

- Vanilla HTML + inline `<style>` + inline `<script>`
- Tailwind Play CDN
- Phosphor Icons (regular + fill + bold)
- Inter via Google Fonts
- Agentation@3 dev-only feedback widget (esm.sh module)

No bundler. Open in any static server.

---

## 2. State machine

Single state attribute on `<body>`: `data-header-state="active" | "expired" | "nudge"`. CSS hides the two non-matching `[data-state]` blocks.

```js
function setState(s) {
  document.body.dataset.headerState = s;
  buttons.forEach(b => b.classList.toggle('is-active', b.dataset.toggle === s));
}
```

### Visibility rules (CSS)

```css
body[data-header-state="expired"] [data-state="active"],
body[data-header-state="expired"] [data-state="nudge"]    { display: none !important; }
body[data-header-state="nudge"]   [data-state="active"],
body[data-header-state="nudge"]   [data-state="expired"]  { display: none !important; }
body:not([data-header-state="expired"]):not([data-header-state="nudge"]) [data-state="expired"],
body:not([data-header-state="expired"]):not([data-header-state="nudge"]) [data-state="nudge"] { display: none !important; }
```

### Click handlers route back to Active

| Trigger | Resolves to |
|---|---|
| Toggle pill (`data-toggle="..."`) | sets that state |
| `#reactivate-cta` (Expired state) | `setState('active')` |
| `#nudge-cta` (Nudge state Upgrade button) | `setState('active')` |
| `#claim-discount-btn` (Nudge state inline link) | `setState('active')` |

---

## 3. State 1 — ACTIVE

```
[$60 off · Ends in 59:51]  [🔥 Trial: 4 days left  Upgrade]
```

### Markup
```html
<div data-state="active" class="ds-state flex items-center gap-3">
  <span class="up-discount-chip" id="up-discount-chip" aria-live="polite">
    <span>$60 off</span>
    <i class="ph-fill ph-clock"></i>
    <span>Ends in <span class="timer" id="up-discount-timer">59:51</span></span>
  </span>
  <div class="trial-chip-wrap bg-white rounded-md flex items-center pl-2.5 pr-1 py-1">
    <div class="trial-chip-stack">
      <div class="trial-chip-track">
        <span>🔥 Trial: 4 days left</span>
        <span>✨ Credits: 60 left</span>
        <span>🔥 Trial: 4 days left</span>
      </div>
    </div>
    <button class="shimmer-cta ml-3 ...">
      <i class="ph-fill ph-rocket-launch"></i>
      Upgrade
    </button>
  </div>
</div>
```

### Animations
- **`up-chip-bob`** 2.6s ∞ — `.up-discount-chip` vertical bob (-3px)
- **`up-chip-glow`** 2.2s ∞ — ring `0 → 9px rgba(255,85,0,0.55→0)` ripple
- **`trial-loop`** ~6s ∞ — Trial / Credits chip cycler in `.trial-chip-track` (vertical translate stack)
- **`shimmer-sweep`** 3s ∞ — diagonal white sweep across orange `Upgrade` button via `::after`

### Styles
- `.up-discount-chip` — orange `#ff5500` pill, 11px/800/0.06em letter-spacing, white text, shadow `0 4px 14px rgba(0,0,0,0.18)`
- `.up-discount-chip::after` — subtle outer glow ring synced with bob
- `.trial-chip-wrap` — white pill with hairline border `rgba(255,85,0,0.12)`, height ~36px

---

## 4. State 2 — EXPIRED

```
⚡ Missed $60 off Pro Annual →   [↻ Reactivate and Upgrade]
                                    ↳ tooltip: One reactivation left
                                              Tap to bring back $60 off…
```

### Markup
```html
<div data-state="expired" class="ds-state flex items-center gap-3">
  <span class="text-[14px] font-medium text-ink-primary inline-flex items-center gap-2">
    <span style="font-size:16px;">⚡️</span>
    <span>Missed <strong class="font-semibold">$60 off Pro Annual</strong></span>
    <i class="ph-bold ph-arrow-right text-[16px] text-brand-500 missed-arrow"></i>
  </span>
  <span class="reactivate-wrap">
    <button id="reactivate-cta" class="shimmer-cta ...">
      <i class="ph-bold ph-arrow-clockwise"></i>
      Reactivate and Upgrade
    </button>
    <span class="reactivate-tooltip" role="tooltip">
      <strong>One reactivation left</strong>
      <span class="muted">Tap to bring back $60 off Pro Annual. Last time we can offer it.</span>
    </span>
  </span>
</div>
```

### Animations
- **`missed-arrow-bounce`** 1.2s ∞ — orange arrow bounces between text and CTA, drawing eye to the action (translateX 0 → 4px → 0)
- **`shimmer-sweep`** 3s ∞ — same as Active state on the Reactivate button
- **`reactivate-tooltip`** — fade-in on `.reactivate-wrap:hover` (0.15s ease, opacity 0→1)

### Tooltip
- Dark `#1a1a1a` bg, white text, 12px header bold + 11px muted body
- Arrow `::before` pointing down at button
- Hidden by default `opacity: 0; pointer-events: none`; revealed via parent hover

---

## 5. State 3 — NUDGE (close-feedback)

```
[🎁 Share feedback and get $60 • Claim Discount]   [🔥 Trial: 4 days · Upgrade]
```

### Markup
```html
<div data-state="nudge" class="ds-state flex items-center gap-3">
  <span class="up-discount-chip" id="nudge-discount-chip">
    <span style="font-size:14px;">🎁</span>
    <span>Share feedback and get $60</span>
    <span class="opacity-60">•</span>
    <button id="claim-discount-btn" class="font-bold underline underline-offset-2"
            style="background:transparent; border:0; padding:0; color:inherit; cursor:pointer;">
      Claim Discount
    </button>
  </span>

  <div class="trial-chip-wrap ...">
    <div class="trial-chip-stack">
      <div class="trial-chip-track">
        <span>🔥 Trial: 4 days left</span>
        <span>✨ Credits: 60 left</span>
        <span>🔥 Trial: 4 days left</span>
      </div>
    </div>
    <button id="nudge-cta" class="shimmer-cta ml-3 ...">
      <i class="ph-fill ph-rocket-launch"></i>
      Upgrade
    </button>
  </div>
</div>
```

### Animations
- **`nudge-glow`** 2.4s ∞ — `.up-discount-chip` (nudge variant) gets soft orange ring
- **`up-chip-bob`** 2.6s ∞ — same bob as Active state chip (shared class)
- **`shimmer-sweep`** 3s ∞ on Upgrade button
- **`trial-loop`** ~6s ∞ — Trial/Credits stack cycler

### Inline "Claim Discount" button
- Underlined text, transparent bg, inherits chip color (white)
- Hover: 90% opacity (`hover:opacity-90`)
- Click → `setState('active')`

---

## 6. State swap animation

All `[data-state]` blocks animate in via:

```css
[data-state] {
  animation-duration: 0.35s;
  animation-fill-mode: both;
  animation-timing-function: cubic-bezier(0.22, 1, 0.36, 1);
}
body[data-header-state="active"]  [data-state="active"]  { animation-name: state-fade-in; }
body[data-header-state="expired"] [data-state="expired"] { animation-name: state-fade-in; }
body[data-header-state="nudge"]   [data-state="nudge"]   { animation-name: state-fade-in; }
@keyframes state-fade-in {
  from { opacity: 0; transform: translateY(-2px); }
  to   { opacity: 1; transform: translateY(0); }
}
```

Spring-bounce previously used here was replaced with simple fade-in for cleanliness.

---

## 7. Animation cheat-sheet

| Keyframe | Duration | Selector | Purpose |
|---|---|---|---|
| `up-chip-bob` | 2.6s ∞ | `.up-discount-chip` (active + nudge) | vertical bob -3px |
| `up-chip-glow` | 2.2s ∞ | `.up-discount-chip` (active) | ripple ring 0→9px |
| `nudge-glow` | 2.4s ∞ | `.up-discount-chip` (nudge variant) | softer orange ring |
| `missed-arrow-bounce` | 1.2s ∞ | `.missed-arrow` (expired) | bounce → toward CTA |
| `shimmer-sweep` | 3s ∞ | `.shimmer-cta::after` | diagonal white sweep on orange CTAs |
| `trial-loop` | ~6s ∞ | `.trial-chip-track` | vertical scroll between Trial / Credits |
| `state-fade-in` | 0.35s spring | active/expired/nudge `[data-state]` | state swap entrance |

---

## 8. CSS tokens

| Color | Hex | Usage |
|---|---|---|
| Brand orange | `#ff5500` | All accent CTAs, chip bg |
| Hot orange (border) | `rgba(255,85,0,0.12)` | Trial chip wrap border |
| Ring overlay (chip glow) | `rgba(255,85,0,0.55)` → `0` | `up-chip-glow`, `nudge-glow` |
| Tooltip bg | `#1a1a1a` | `.reactivate-tooltip` |
| Tooltip muted | `#d4d4d4` | tooltip body |
| Ink primary | `#171717` | strong text |
| Ink body | `#525252` | body text |
| White | `#ffffff` | trial chip wrap bg, CTA text |

| Sizing | Value |
|---|---|
| Discount chip | padding 5px 12px / 11px font / radius 999px |
| Reactivate CTA | h-9 / px-5 / 14px font / bold |
| Upgrade CTA (compact) | h-7 / px-3.5 / 14px font / medium |
| Trial-chip wrap | h ~36px / radius 6px |
| Tooltip | width auto / radius 8px / 12px shadow |

---

## 9. JS surface

### Mount-time wiring
```js
const buttons = document.querySelectorAll('.state-toggle button[data-toggle]');
function setState(s) {
  document.body.dataset.headerState = s;
  buttons.forEach(b => b.classList.toggle('is-active', b.dataset.toggle === s));
}
buttons.forEach(b => b.addEventListener('click', () => setState(b.dataset.toggle)));
setState('active'); // default

document.getElementById('reactivate-cta')?.addEventListener('click', () => setState('active'));
document.getElementById('nudge-cta')     ?.addEventListener('click', () => setState('active'));
document.getElementById('claim-discount-btn')?.addEventListener('click', () => setState('active'));
```

### State toggle UI
Bottom-right floating pill with 3 buttons (`data-toggle="active|expired|nudge"`). Active state highlighted via `.is-active` class. Strip for production.

### Countdown
`#up-discount-timer` text is static placeholder (`59:51`). Wire to live `setInterval` that decrements every 1000ms in production.

---

## 10. Production checklist

1. **Strip the bottom-right state toggle** — debug widget. State should be driven by app context (trial expired? feedback dismissed? clock running?).
2. **Wire countdown timer** — `#up-discount-timer` is currently static. Run `setInterval(() => updateTimer(end - Date.now()), 1000)` and clear on expiry → flip state to `expired`.
3. **Persist state across refresh** — store `bumpStart` / `feedbackSubmitted` in `localStorage` keyed by user id.
4. **Wire CTA destinations:**
   - `#reactivate-cta` → checkout with `?promo=REACTIVATE60`
   - `#nudge-cta` (Upgrade) → checkout
   - `#claim-discount-btn` → open close-feedback modal *(testimonial flow from `export-ppt-pricing.html`)*
5. **Strip Agentation block** (dev-only widget at end of file).
6. **A11y audit:**
   - `.up-discount-chip` already has `aria-live="polite"` — keep
   - Phosphor icons need `aria-hidden="true"`
   - Tooltip should be keyboard-reachable (`tabindex="0"` on `.reactivate-wrap` button)
   - `prefers-reduced-motion` media query to disable bob/glow/shimmer/missed-arrow
7. **Mobile** — currently desktop-only nav strip. Add stacking media query OR compact icons-only view at <768px.

---

## 11. Versioning

| Date | Change |
|---|---|
| 2026-04-29 | Initial scaffold from `dashboard-flash-sale.html`. 3-state CSS visibility system. |
| 2026-04-29 | Reactivate hover tooltip ("One reactivation left"). Spring-bounce → simple fade-in. |
| 2026-04-29 | "· Last chance" text replaced with animated `missed-arrow` bouncing right. |
| 2026-04-29 | Close-nudge state added — Help-us-improve chip + Take-survey CTA. |
| 2026-04-29 | Removed arrow icon from nudge state. |
| 2026-04-29 | Nudge state restructured: chip becomes `Help us improve and get $60 • Claim Discount` (underlined inline button) + Trial-chip wrap with Upgrade rocket CTA (matches active state pattern). |
| 2026-05-05 | Nudge chip copy: "Help us improve and get $60" → **"Share feedback and get $60"**. |
