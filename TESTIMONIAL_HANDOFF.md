# Testimonial Modal — Engineering Handoff

**File:** `testimonial.html` (~1,000 lines, single self-contained HTML)
**Purpose:** Close-intent feedback popup that swaps the legacy 2-question survey ("Rate / Improve") with a 1-tap **emoji → testimonial** picker. User taps a feeling, system writes a marketing-ready testimonial; or user types their own. Both paths unlock $60 off Pro Annual.

Two render modes ship in the same file behind a top-center toggle:

| Mode | Layout | When to use |
|------|--------|-------------|
| **Testimonial** *(default)* | Single-column, orange header on top + body below. 520px wide. | New design — replaces the existing close-intent survey. |
| **V2** | Two-column, orange `.fb-panel-left` left + form right. 620px wide. | Drop-in compatible with v2.html `FeedbackModal` shell. |

Both modes share the same picker logic, FEELINGS data, and submit flow.

---

## 1. Tech stack

- Vanilla HTML + inline `<style>` + inline `<script>` — no build step.
- Phosphor Icons (regular / fill / bold) via CDN.
- Inter (400/500/600/700/800) + Instrument Serif (italic) via Google Fonts.
- Agentation@3 dev-only feedback widget mounted via esm.sh module script (line ~1010).
- No frameworks, no bundler. Open file directly in browser or serve via any static server.

---

## 2. File anatomy

```
testimonial.html
├─ <head>
│   ├─ Phosphor + Google Fonts <link>s
│   └─ <style> ~600 lines
│       ├─ Reset + body
│       ├─ Modal backdrop
│       ├─ Mode toggle (top-center pills)
│       ├─ TESTIMONIAL MODE styles (.offer-nudge, .nudge-header, .testimonial, .input-card, .emoji-row, .emoji-track, .emoji-chip, .cta, .cta-lock)
│       ├─ V2 MODE styles (.fb-card, .fb-survey-wrap, .fb-panel-left, .fb-panel-title, .fb-tag, .fb-conf, .fb-right-col, .fb-btn-primary, .fb-lock)
│       ├─ Success step (.fb-success-title, .fb-success-body, .fb-cta-wrap, .fb-cta-badge)
│       └─ @keyframes (gift-bounce, conf-drift, emoji-marquee, lock-pop, cta-shine, fade-in, fb-arrow-pulse, fb-cta-badge-pulse, nudge-pop)
├─ <body>
│   ├─ .mode-toggle (fixed top-center)
│   ├─ .modal-backdrop
│   │   ├─ [data-modal-shell="testimonial"] (default visible)
│   │   │   └─ .offer-nudge (520×auto)
│   │   │       ├─ .close-btn (top-right, outside)
│   │   │       ├─ .nudge-header (orange, 140px, gift + badge + price + 10 confetti)
│   │   │       └─ .testimonial
│   │   │           ├─ [data-step="form"] heading + input-card + disclaimer + cta
│   │   │           └─ [data-step="success"] thank-you + countdown badge + claim CTA
│   │   └─ [data-modal-shell="v2"] (hidden by default)
│   │       └─ .fb-dialog-wrap
│   │           ├─ .fb-close (top-right, outside)
│   │           └─ .fb-card → .fb-survey-wrap
│   │               ├─ .fb-panel-left (orange, gift + tag + price + 5 confetti)
│   │               └─ .fb-right-col
│   │                   ├─ [data-step="form"] heading + input-card + disclaimer + submit
│   │                   └─ [data-step="success"] same as testimonial
│   ├─ <script> ~280 lines (FEELINGS + wireInputCard + showSuccess + toggle handlers)
│   └─ Agentation mount
```

---

## 3. Layout

### Testimonial mode (default)

```
┌─────────────────────────────────────────┐
│ ╭─ orange gradient header (140px) ─╮  ✕ │
│ │ 🎁  HELP US IMPROVE AND GET       │   │
│ │     $60                            │   │
│ │     off Pro Annual                 │   │
│ ╰────────────────────────────────────╯   │
│                                          │
│  Pick a feeling - we'll write the       │
│  testimonial                             │
│                                          │
│  ┌────────────────────────────────────┐ │
│  │  In just 2 minutes - I was able... │ │
│  │                                    │ │
│  │  [😀 Happy] [⚡ Fast] [📝 Detail.. │ │ ← marquee R→L
│  └────────────────────────────────────┘ │
│                                          │
│  We may share your testimonial...        │
│                                          │
│  [🔒 Submit to unlock discount]          │
└─────────────────────────────────────────┘
```

- **Width:** 520px (max-width: 100vw - 48px)
- **Modal:** rounded-12px white, drop shadow `0 30px 60px -12px rgba(0,0,0,0.35)`
- **Header gradient:** `linear-gradient(164.93deg, #ffdcbf 0%, #fff3e6 71.43%)`
- **Body padding:** `24px 32px 28px`, gap 12px between rows.
- **Close button:** absolute top:-44 right:-44 outside the card (40px white circle, shadow). On viewport <600 it tucks back inside (top:8 right:8).

### V2 mode

```
┌────────────────────────────────────────────┐
│ ┌──────────┬─────────────────────────────┐ ✕│
│ │  🎁      │  Pick a feeling…           │  │
│ │ HELP US…│                              │  │
│ │  $60     │  ┌────────────────────────┐ │  │
│ │ off Pro  │  │ marquee + text field   │ │  │
│ │ Annual   │  └────────────────────────┘ │  │
│ │          │  We may share…              │  │
│ │ confetti │  [🔒 Submit to unlock disc.]│  │
│ └──────────┴─────────────────────────────┘  │
└────────────────────────────────────────────┘
```

- **Width:** 620px, two-column grid `240px 1fr`.
- **Left panel:** `#ffeee5` peach with bottom-left radial gradient `rgba(255,85,0,0.22)` and 5 confetti pieces.
- **Right col padding:** `32px 32px 28px`.

---

## 4. Mode toggle

```html
<div class="mode-toggle">
  <button class="on" data-mode="testimonial">Testimonial</button>
  <button data-mode="v2">V2</button>
</div>
```

- Fixed `top:16px left:50% translateX(-50%)` z-index 200, white pill with shadow.
- Active button: `background:#0A1925 color:#fff`. Inactive: gray text.
- JS toggles `[data-modal-shell].on` (display: flex) and re-focuses the active text field via `focusActiveQuote()`.
- Removed for production: this is a debug switcher. Pick one mode, delete the other shell + the `.mode-toggle` markup.

---

## 5. Picker card (`.input-card`)

Shared between both modes via `wireInputCard(cardId, trackId, quoteId, ctaId)`.

### Markup
```html
<div class="input-card" id="input-card">
  <div class="quote" contenteditable="true" role="textbox" aria-multiline="true"
       data-placeholder="Tap a feeling — or type your own testimonial."></div>
  <div class="emoji-row">
    <div class="emoji-track" id="emoji-track"></div>
  </div>
</div>
```

### Behavior

| User action | Result |
|---|---|
| Page load | `quote` empty → placeholder visible. Marquee scrolling. CTA disabled with lock icon. |
| Tap emoji chip | Active state syncs across both copies of the chip in the marquee track. Random quote from `FEELING_BY_LABEL[label].qs[]` fills `quote`. `.input-card.picked` toggles → marquee fades out + slides 8px down. CTA unlocks (`lock-pop` + `cta-shine`). |
| Type in quote | Same as picking: `.picked` toggles on, marquee hides, CTA unlocks on first char. |
| Backspace to empty | `.picked` removed, marquee fades back in, CTA re-disables. |
| Click submit | `data-step="form"` hides → `data-step="success"` shows (fade-in 0.35s). Tag swaps `Help us improve and get` → `You've unlocked`. 60-min countdown starts. |

### Styles

- **Card:** `175px` tall, border `rgba(26,26,26,0.15)`, radius `4px`, padding `19px`, bg `#fafafa`. Focus-within: border `#0A1925`, bg `#fff`.
- **Quote:** font 13px, line-height 1.5, min-height 60px / max-height 80px (overflow-y auto, custom 4px scrollbar). When `.picked`: min-height 130px, max-height 130px.
- **Marquee viewport (`.emoji-row`):** absolute bottom:8px, height 70px, overflow visible (chips lift on hover without clipping). Pseudo `::before`/`::after` gradient fades in/out at left/right edges to mask chip entry/exit.

---

## 6. Emoji marquee chips

### Markup (built dynamically)
```js
const buildChip = (f) => {
  const b = document.createElement('button');
  b.className = 'emoji-chip';
  b.dataset.feeling = f.label;
  b.title = f.label;
  b.innerHTML = `<span class="em">${f.em}</span><span class="lbl">${f.label}</span>`;
  return b;
};
// Doubled for seamless loop:
FEELINGS.forEach((f) => track.appendChild(buildChip(f)));
FEELINGS.forEach((f) => track.appendChild(buildChip(f)));
```

### Styles

- Padding `6px 12px`, border `1px rgba(26,26,26,0.15)`, radius `4px`, font 13px/500 `#525252`.
- **Hover:** border `#171717`, color `#171717`, `transform: rotate(-3deg) scale(1.06) translateY(-2px)`, shadow lifts. z-index 2.
- **Active:** border `#0A1925`, bg `#f2f5f9`, color `#0A1925`.
- **Marquee:** `animation: emoji-marquee 60s linear infinite` on `.emoji-track`. `from { translate3d(0,0,0) }` → `to { translate3d(-50%, 0, 0) }`. Pause on `.emoji-row:hover`.
- **Marquee viewport:** sits flush at `bottom: 0` of the input card.

---

## 7. Data model — `FEELINGS`

10 feelings × 5 testimonials each = **50 random presentation-themed quotes**. All quotes talk about decks, slides, board updates, investor pitches, QBRs, keynotes, brand-voice, narrative, etc.

```js
const FEELINGS = [
  { em: "😀", label: "Happy",      qs: [/* 5 quotes */] },
  { em: "⚡️", label: "Fast",        qs: [/* 5 quotes */] },
  { em: "📝", label: "Detailed",    qs: [/* 5 quotes */] },
  { em: "🤯", label: "Mind blown",  qs: [/* 5 quotes */] },
  { em: "😍", label: "Love it",     qs: [/* 5 quotes */] },
  { em: "😲", label: "Surprised",   qs: [/* 5 quotes */] },
  { em: "🙌", label: "Grateful",    qs: [/* 5 quotes */] },
  { em: "🖌️", label: "Crafted",     qs: [/* 5 quotes */] },
  { em: "🤔", label: "Mixed",       qs: [/* 5 quotes */] },  // negative-leaning
  { em: "😬", label: "Rough",       qs: [/* 5 quotes */] },  // negative
];
```

`pickRandomQuote(label)` returns a random quote from the matching feeling's `qs[]`. Re-tapping the same chip yields a different quote each time.

**Edit / extend:** add/remove entries in the array. Marquee + V2 picker auto-update on next page load.

---

## 8. Submit → success step

Both modes have `[data-step="form"]` (default `.on`) and `[data-step="success"]` (hidden) inside their right-col / body. CSS:

```css
[data-step] { display: none; }
[data-step].on { display: block; animation: fade-in 0.35s ease; }
```

### Success markup
```html
<div data-step="success" data-shell="testimonial">
  <h2 class="fb-success-title">Thank you</h2>
  <p class="fb-success-body">
    We heard you. As a token of appreciation, we've unlocked
    <strong>$60 off Pro Annual</strong> for the next <strong>one hour</strong>.
  </p>
  <div class="fb-actions fb-cta-wrap">
    <span class="fb-cta-badge">
      <i class="ph-fill ph-clock"></i>
      Ends in <span class="fb-btn-timer" data-timer>59:59</span>
    </span>
    <button class="fb-btn-primary" data-claim>Buy Pro Annual at $60 off</button>
  </div>
</div>
```

### `showSuccess(shellName)` — JS

1. Toggles `.on` from `[data-step="form"]` → `[data-step="success"]`.
2. Swaps tag copy: testimonial mode `.badge` text or v2 mode `.fb-tag` text → `"You've unlocked"`.
3. Starts 60-minute countdown via `startTimer(el, 60*60-1)` — `setInterval` 1000ms updating `MM:SS`.

The pulsing orange `.fb-cta-badge` floats above the CTA (animation `fb-cta-badge-pulse 1.8s ease-in-out infinite`).

---

## 8b. Price reveal — skeleton → odometer roll

The `$60` amount in both modes simulates a server fetch on page load:

```html
<span class="amt" data-price-loading="true" data-price-from="40" data-price-target="60">
  <span class="amt-skeleton" aria-hidden="true"></span>
  <span class="amt-real"><span class="sym">$</span><span class="amt-num">60</span></span>
</span>
```

### Sequence

| Phase | Duration | What happens |
|---|---|---|
| Loading | 0 → 2000ms | `.amt-skeleton` shows: 92×36 peach gradient block, shimmer (`skeleton-shimmer 1.1s ∞`). `.amt-real` is `display: none`. |
| Reveal | 2000 → 2250ms | JS flips `data-price-loading="false"`. Skeleton hides, real fades in (`price-fade-in 0.25s`). |
| Roll | 2000 → 3000ms | `rollNumber()` walks `.amt-num` from `data-price-from` → `data-price-target` over 1000ms with cubic ease-out. Default 40 → 60. |

### Configuration (per `.amt` element)

- `data-price-from="40"` — start value (default 40 in JS fallback)
- `data-price-target="60"` — end value (default 60)
- `data-price-loading="true|false"` — toggled by JS at the 2s mark

To change the delay, edit the `setTimeout(..., 2000)` in the `<script>` block.

### Why the gradient lives on `.amt-real`, not `.amt`

`.amt` is `display: inline-flex` with two children (skeleton + real). Putting `background-clip: text` on `.amt` broke the gradient propagation when text was wrapped in `.amt-real .amt-num`. The gradient is applied directly to `.amt-real` so the inner `.sym` + `.amt-num` spans inherit transparent fill and the gradient clips correctly to text shape.

```css
.price-line .amt      { color: #ff5500; }            /* fallback */
.price-line .amt-real {
  background: linear-gradient(180deg, #ff732d 0%, #ff5500 100%);
  -webkit-background-clip: text; background-clip: text;
  color: transparent;
}
```

Same pattern repeats for `.fb-panel-title .amt-real` in V2 mode.

---

## 9. CTA — lock-pop + shine

Identical visual + behavior in both modes (testimonial mode uses `.cta`, v2 uses `.fb-btn-primary` — same gradient, same lock pattern):

```html
<button class="cta" disabled>
  <span class="cta-lock">
    <i class="ph-fill ph-lock"></i>      <!-- visible while disabled -->
    <i class="ph-fill ph-lock-open"></i> <!-- visible when enabled -->
  </span>
  <span>Submit to unlock discount</span>
</button>
```

- **Background:** `linear-gradient(180deg, #1c3550 0%, #0A1925 100%)`. Disabled: `#e5e5e5` / `#a3a3a3`.
- **`lock-pop` keyframe** (0.55s spring): -18deg/0.6 → 8deg/1.15 → 0deg/1.0 — fires when `.cta:not([disabled])` flips on first emoji pick or first typed character.
- **`cta-shine` keyframe** (0.75s linear): white sweep from 150% → -50% on `.just-unlocked` class. Re-triggered each time CTA goes from disabled → enabled (forced reflow `void cta.offsetWidth`).

---

## 10. Header confetti

Static `<span class="conf">` pieces with absolute positioning + per-piece CSS variables `--r`, `--tx`, `--ty`. Animation: `conf-drift 4.6s ease-in-out infinite` with per-piece `animation-duration` (4.2s–6s) + `animation-delay` (-0.3s to -2.7s) for natural cadence.

```css
@keyframes conf-drift {
  0%, 100% { transform: translate(0,0) rotate(var(--r,0deg)); }
  50%      { transform: translate(var(--tx,0), var(--ty,-10px))
                        rotate(calc(var(--r,0deg) + 12deg)); }
}
```

- **Testimonial header:** 10 pieces spread across full 520px width, top + bottom edges, avoiding gift/copy zone (left:30 + ~140-300).
- **V2 panel-left:** 5 pieces on right side of the 240px column.

Mix of rect chips (rotated) + circle dots in 5 colors: `#ff5500`, `#ff732d`, `#ff801a`, `#4285f4`, `#16a34a`, `#fbbc04`, `#f24000`.

---

## 11. Animations cheat-sheet

| Keyframe | Duration | Where | Purpose |
|----------|----------|-------|---------|
| `gift-bounce` | 1.8s ∞ | `.gift-emoji`, `.fb-gift` | 🎁 bobs up/down |
| `conf-drift` | 4.2–6s ∞ | `.conf`, `.fb-conf` | confetti drift + rotate |
| `emoji-marquee` | 60s linear ∞ | `.emoji-track` | right-to-left infinite scroll (slow) |
| `skeleton-shimmer` | 1.1s ∞ | `.amt-skeleton` | peach gradient sweep during 2s load |
| `price-fade-in` | 0.25s ease-out | `.amt[data-price-loading="false"] .amt-real` | $XX fades in after skeleton |
| `lock-pop` | 0.55s spring | `.cta-lock .ph-lock-open` | padlock unlocks on enable |
| `cta-shine` | 0.75s linear | `.cta.just-unlocked::before` | white sweep across CTA |
| `fb-arrow-pulse` | 1.4s ∞ | `.fb-panel-sub .arrow` | rightward nudge (V2 left panel) |
| `fb-cta-badge-pulse` | 1.8s ∞ | `.fb-cta-badge` | success timer badge bobs |
| `nudge-pop` | 0.45s spring | `.offer-nudge` | modal scale-in on mount |
| `fade-in` | 0.35s ease | `[data-step].on` | form → success transition |
| `rollNumber` (JS) | 1000ms ease-out | `.amt-num` | digit count from `from` → `to` |

---

## 12. CSS tokens

| Color | Hex | Usage |
|-------|-----|-------|
| Brand orange | `#ff5500` | gradient stops, accents, badge bg |
| Hot orange | `#ff732d` | gradient mid, confetti |
| Light orange | `#ff801a` | confetti |
| Tag bg | `rgba(255,85,0,0.22)` | `.fb-tag` |
| Tag text | `#c64200` | `.fb-tag` color |
| Header gradient | `#ffdcbf → #fff3e6` | `.nudge-header` 164.93deg |
| Peach panel | `#ffeee5` | `.fb-panel-left` bg |
| Ink primary | `#171717` | strong text |
| Ink body | `#525252` | body text |
| Ink mute | `#8d8d8d` | disclaimer |
| Card bg | `#fafafa` | `.input-card` idle |
| Card border | `rgba(26,26,26,0.15)` | input-card + chips |
| Active stroke | `#0A1925` | active chip + focused card |
| Active fill | `#f2f5f9` | active chip bg |
| CTA gradient | `#1c3550 → #0A1925` | `.cta`, `.fb-btn-primary` |
| CTA disabled | `#e5e5e5` / `#a3a3a3` | bg / text |
| Confetti | `#4285f4`, `#16a34a`, `#fbbc04`, `#f24000` | dots |

| Spacing | Value |
|---------|-------|
| Modal radius | 12px (testimonial), 4px (v2) |
| Card radius | 4px |
| Chip radius | 4px |
| CTA radius | 4px |
| Body padding | 24/32/28 (testimonial), 32/32/28 (v2 right col) |
| Gap between rows | 12px |
| Header height | 140px (both modes) |

---

## 13. JS surface

### Public functions
None. Everything is IIFE-scoped.

### Internal helpers
- `buildChip(f)` — build single emoji button.
- `pickRandomQuote(label)` — random quote from `FEELING_BY_LABEL[label].qs`.
- `wireInputCard(cardId, trackId, quoteId, ctaId)` — wires marquee chips + contenteditable input + CTA enable logic for one card.
- `triggerUnlock()` — flips CTA `disabled = false`, restarts `cta-shine` via reflow.
- `onPick(chip)` — sets active class on both copies of the chip, fills quote, marks card `.picked`, fires `triggerUnlock()`.
- `startTimer(el, totalSec)` — countdown helper, `setInterval` 1000ms.
- `showSuccess(shellName)` — swaps step from form → success, swaps tag copy, kicks off countdown.
- `focusActiveQuote()` — auto-focus contenteditable on load + on toggle switch.
- `rollNumber(el, from, to, dur)` — odometer-style digit count using `requestAnimationFrame`, cubic ease-out.

### Wiring at boot
```js
wireInputCard('input-card',    'emoji-track',    'quote',    'cta');       // testimonial mode
wireInputCard('input-card-v2', 'emoji-track-v2', 'quote-v2', 'fb-submit'); // v2 mode

document.getElementById('cta')      .addEventListener('click', () => showSuccess('testimonial'));
document.getElementById('fb-submit').addEventListener('click', () => showSuccess('v2'));

document.querySelectorAll('.mode-toggle button').forEach((btn) => {
  btn.addEventListener('click', () => { /* swap [data-modal-shell].on */ });
});
```

---

## 14. Mobile

- **Backdrop:** `padding: 24px` so the modal never touches viewport edges.
- **Modal max-width:** `100%` + `max-width: calc(100vw - 48px)` on `.fb-card`. Testimonial `.offer-nudge`: `max-width: 100%`.
- **Close button:** at viewport <600 it moves from outside the modal (`top:-44 right:-44`) to inside (`top:8 right:8`).
- **V2 mode at <600:** the 240px peach left column will need either a stacked-flex media query or fall back to testimonial mode. Currently no media query — left as a TODO if mobile V2 is required.

---

## 15. Production checklist

Before shipping:

1. **Pick one mode** — delete the other `[data-modal-shell]` block and the `.mode-toggle` markup. Toggle is debug-only.
2. **Wire submit to backend** — `document.getElementById('cta')` (or `'fb-submit'`) click currently only swaps to success step locally. POST to `/api/feedback` with `{ feeling, quote, opted_in_to_share }`.
3. **Persist discount** — currently the 60-min timer is UI-only. Store the bumped offer in `localStorage` or backend session so refresh doesn't lose it (mirror v2's `bumpStart` pattern).
4. **Wire the claim CTA** — `[data-claim]` button currently no-op. Should route to checkout with `?promo=PROANNUAL60`.
5. **Strip Agentation block** (line ~1010) — dev-only widget, do not ship to production.
6. **Strip auto-focus** if you don't want the cursor to jump into the field on mount (line ~770 `focusActiveQuote()`).
7. **Localize:** all visible copy in HTML + the `qs[]` arrays in `FEELINGS`.
8. **Telemetry:** log feeling chosen + whether quote was edited vs. picked verbatim — useful diagnostic data the legacy 2-question survey gave you.
9. **Optional consent gate:** disclaimer says "We may share your testimonial..." — for stricter consent, replace with an opt-in checkbox before publishing.
10. **Image fallback:** Phosphor icons load via CDN; pin a version (`@2.1.1`) is already done.
11. **A11y audit:** `role="textbox"` + `aria-multiline` set. Add `aria-label` on chips, ensure focus order from quote → CTA.

---

## 16. How to integrate

### Drop-in replacement for v2.html FeedbackModal

Use **V2 mode**. Copy the `[data-modal-shell="v2"]` block + the `.fb-card`, `.fb-survey-wrap`, `.fb-panel-*`, `.fb-conf`, `.fb-right-col`, `.fb-btn-primary`, `.fb-lock`, `.fb-success-*`, `.fb-cta-*` CSS rules into `v2.html`. Replace the existing `<div id="fb-step-form">` / `<div id="fb-step-success">` survey content with the new picker block + the same success step. The `wireInputCard()` JS handles the rest.

### Standalone modal (recommended for new flows)

Use **Testimonial mode**. Copy `testimonial.html` as-is, strip the V2 shell + toggle, and:
- Trigger the modal on close-intent (mouseleave from window, or a "Cancel subscription"-type CTA).
- Listen for `cta` click to fire your conversion event.

### Embedded inline (no backdrop)

Wrap `.offer-nudge` in your own container, drop `.modal-backdrop` and `.close-btn`. Modal is otherwise self-contained.

---

## 17. Key files referenced

- `testimonial.html` — this file.
- `v2.html` — original close-nudge modal whose styles + success step were ported.
- `dashboard-flash-sale-expired.html` — companion dashboard variant that triggers this modal from the "Claim Discount" link.
- `_archive/` — older variants kept for reference.

---

## 18. Versioning

| Date | Change |
|------|--------|
| 2026-04-29 | Initial scaffold ported from `crazy8s.html` Round 1 V1. |
| 2026-04-30 | Added v2 toggle mode, full v2 popup parity, success step + countdown, expanded confetti, contenteditable text field, 50 randomized presentation testimonials, lock-pop CTA, infinite marquee. Handoff doc written. |
| 2026-04-30 | Price reveal: 2s peach skeleton → odometer roll `$40 → $60` (configurable via `data-price-from` / `data-price-target`) + 0.25s fade-in. Marquee speed slowed `36s → 60s`. Marquee row aligned flush at `bottom: 0`. Removed earlier scale-pop / blur reveal — just clean fade + roll. Gradient `background-clip:text` moved from `.amt` → `.amt-real` to fix invisible-digits bug. |
