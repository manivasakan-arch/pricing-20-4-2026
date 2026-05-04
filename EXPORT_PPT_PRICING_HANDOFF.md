# Export PPT Pricing Modal — Engineering Handoff

**File:** `export-ppt-pricing.html` (~1700 lines, single self-contained HTML)

**Purpose:** Modal-over-dashboard pricing screen. User finishes a deck → sees this modal with two cards (Single PPT one-time + Pro Plan annual) and a testimonial panel. Close-intent triggers a feedback modal that, on submit OR close, applies a deeper discount to the Pro Plan card.

Three audience modes (top-center toggle, debug only — pick one for production):

| Mode | Pricing on load | Notes |
|---|---|---|
| **Tier 1 — Offer not seen** | Single PPT $119 · Pro Plan $20/mo | Default. No nudge chip. |
| **Tier 1 — Offer seen** | Single PPT $119 · Pro Plan **$15/mo** (strikeout $20) | Bumped already. Pulsing chip + 60-min countdown. No close-feedback flow needed. |
| **EDU** | Single PPT $59 · Pro Plan **$15/mo** (strikeout $20, "25% education discount") | Default state. Close X → feedback modal → bumped to **$10/mo** ("50% student discount"). |

---

## 1. Tech stack

- React 18 via esm.sh (no bundler, importmap)
- Babel standalone for in-browser JSX compile
- Tailwind Play CDN
- Phosphor Icons (regular + fill + bold) via CDN
- Inter (400/500/600/700/800) + Hedvig Letters Serif + Instrument Serif via Google Fonts
- Agentation@3 dev-only feedback widget mounted via esm.sh module script

No bundler. Open file directly in any static server.

---

## 2. Pricing data model

### Constants

```js
// EDU Single PPT — never discounts (oneTime locked to BASE in derivation)
const ONE_TIME_BASE = {
  shortName: "Single PPT",
  originalPrice: 59, price: 59,
  hideStrikeout: true,
  discountLabel: null,
  buyLabel: "Export a single PPT",
  compareDiscount: "Single export",
};
const ONE_TIME_BUMPED = { /* unused — Single PPT never bumps */ };

// EDU Pro Plan
const UNLIMITED_BASE = {
  shortName: "Pro Plan",
  originalPriceMonthly: 20,
  studentPriceMonthly: 15,                // 25% off
  discountLabel: "25% education discount",
  buyLabel: "Buy Now • 25% Off",
  billed: "billed annually",
};
const UNLIMITED_BUMPED = {
  ...UNLIMITED_BASE,
  studentPriceMonthly: 10,                // 50% off (after close-feedback)
  discountLabel: "50% student discount",
  buyLabel: "Buy Now • 50% Off",
};

// Tier 1 Single PPT
const ONE_TIME_TIER1_BASE = {
  shortName: "Single PPT",
  originalPrice: 119, price: 119,
  hideStrikeout: true,
  discountLabel: null,
  buyLabel: "Export a single PPT",
};
const ONE_TIME_TIER1_BUMPED = { /* unused */ };

// Tier 1 Pro Plan
const UNLIMITED_TIER1_BASE = {
  shortName: "Pro Plan",
  originalPriceMonthly: 20,
  studentPriceMonthly: 20,
  hideStrikeout: true,
  discountLabel: null,
  buyLabel: "Export unlimited PPT with Pro",
  billed: "billed annually",
};
const UNLIMITED_TIER1_BUMPED = {
  ...UNLIMITED_TIER1_BASE,
  studentPriceMonthly: 15,                // $5/mo off → $60/yr off
  hideStrikeout: false,
  discountLabel: "$60 off Pro Annual",
};
```

### Discount summary table

| Audience | State | Single PPT | Pro Plan / mo | Pro Plan strikeout | Pro discount label | Nudge chip text |
|---|---|---|---|---|---|---|
| Tier 1 | Offer not seen (BASE) | $119 | $20 | none | — | (no chip) |
| Tier 1 | Offer seen (BUMPED) | $119 | **$15** | ~~$20~~ | "$60 off Pro Annual" | **$60 off** · Ends in MM:SS |
| EDU | BASE | $59 | $15 | ~~$20~~ | "25% education discount" | (no chip until close) |
| EDU | BUMPED (after close) | $59 | **$10** | ~~$20~~ | "50% student discount" | **50% off** · Ends in MM:SS |

`extraOneTime` = (always 0 since Single PPT never bumps; computed for legacy code)
`extraUnlimited` (Tier 1 bumped) = `(20 - 15) × 12 = $60` annual
`extraUnlimited` (EDU bumped) = `(15 - 10) × 12 = $60` annual

### Derivation in `Page()`

```js
// Single PPT NEVER discounts — always base across all tabs
const oneTime = audience === "tier1" ? ONE_TIME_TIER1_BASE : ONE_TIME_BASE;

// Pro Plan flips between BASE / BUMPED
const unlimited = audience === "tier1"
  ? (discountTier === "bumped" ? UNLIMITED_TIER1_BUMPED : UNLIMITED_TIER1_BASE)
  : (discountTier === "bumped" ? UNLIMITED_BUMPED : UNLIMITED_BASE);

const testimonial = audience === "tier1" ? TESTIMONIAL_TIER1 : TESTIMONIAL;
```

### Audience toggle (debug only)

```jsx
{[
  { key: "tier1",     label: "Offer not seen" },
  { key: "tier1seen", label: "Offer seen"     },
  { key: "edu",       label: "EDU"            },
]}
```

`tier1seen` click handler bumps Tier1 directly without feedback flow:
```js
setAudience("tier1");
setDiscountTier("bumped");
setBumpStart(Date.now());
setNow(Date.now());
setFeedbackSubmitted(true);
setFeedbackOpen(false);
```

**Production: strip the toggle.** Pick one mode per user segment (auth state, GrowthBook flag, A/B id).

---

## 3. Card layout (CardsImagePlan)

```
┌────────────────────────────┐  ┌────────────────────────────┐
│ illus (.illus-basic)       │  │ illus (.illus-pro)         │
│                            │  │                  [most pop]│
│ Single PPT     ┌─────────┐ │  │ Pro Plan        ┌─────────┐│
│                │ $59     │ │  │                 │~$20~    ││
│                │billed   │ │  │                 │ $15 /mo ││
│                │ once    │ │  │                 │billed   ││
│                └─────────┘ │  │                 │annually ││
│ [Export a single PPT]      │  │ [Buy Now • 25% Off]        │
│                            │  │   ╭──── nudge chip ────╮   │
│                            │  │   │ ⚡ 50% off ·       │   │
│                            │  │   │ Ends in 59:51      │   │
│                            │  │   ╰────────────────────╯   │
│ ✏️ Export just this deck   │  │ 🔖 25% education discount  │
│ ✏️ Pixel-perfect, fully   │  │ ✏️ Export unlimited decks  │
│    editable export        │  │ ✏️ Pixel-perfect, fully   │
│                            │  │ 🪙 5,000 Credits           │
│                            │  │ ⭐ Advanced AI models      │
└────────────────────────────┘  └────────────────────────────┘
```

### Title-row alignment

Both cards: `min-h-[68px]` reserved on title-row + price column. Pro's 3-line price (strikeout / price/mo / billed) and Single's 2-line price both occupy 68px → titles + buttons align horizontally across cards.

### Buttons

Both `h-12 rounded text-[14px] font-bold`:

| Card | State | Label |
|---|---|---|
| Single PPT | All tabs | **"Export a single PPT"** |
| Pro Plan | EDU base | "Buy Now • 25% Off" |
| Pro Plan | EDU bumped | "Buy Now" *(suffix stripped via regex `replace(/\s*•.*$/, '')` when nudge active)* |
| Pro Plan | Tier 1 base | "Export unlimited PPT with Pro" |
| Pro Plan | Tier 1 bumped | "Export unlimited PPT with Pro" *(no `• Off` suffix in source)* |

Single PPT (`OutlinedCta`): white bg, `#0A1925` border + text.
Pro Plan (`CardCta`): navy gradient `linear-gradient(180deg, #1c3550 0%, #0A1925 100%)`, white text.

### Feature list

```js
buildOneTimeFeatures(oneTime) = [
  { icon: "ph-fill ph-seal-percent", text: oneTime.discountLabel, accent: true },  // hidden if null
  { icon: "ph ph-file-ppt",          text: "Export just this deck" },
  { icon: "ph ph-pencil-simple",     text: "Pixel-perfect, fully editable export" },
].filter(f => f.text);

buildUnlimitedFeatures(unlimited) = [
  { icon: "ph-fill ph-seal-percent", text: unlimited.discountLabel, accent: true },
  { icon: "ph ph-file-ppt",          text: "Export unlimited decks" },
  { icon: "ph ph-pencil-simple",     text: "Pixel-perfect, fully editable export" },
  { icon: "ph ph-coin",              text: "5,000 Credits" },
  { icon: "ph ph-star-four",         text: "Advanced AI models and agents" },
].filter(f => f.text);
```

Accent rows: green text `#16a34a`, spinning `.chip-icon-spin` on the seal-percent.

---

## 4. Nudge chip (`.t-nudge-cta-badge`)

Floats below Pro Plan Buy button when `discountTier === "bumped"`. Centered, arrow pointing up, orange.

| Mode | Label |
|---|---|
| EDU | `50% off` |
| Tier 1 | `$60 off` |

Markup:
```jsx
<NudgeChip label={proNudgeLabel} timer={timerLabel} />
```

```css
.t-nudge-cta-badge {
  position: absolute; top: calc(100% + 6px); left: 50%;
  padding: 5px 12px;
  background: #ff5500; color: #fff;
  border-radius: 999px;
  font-size: 11px; font-weight: 800; letter-spacing: 0.06em;
  font-variant-numeric: tabular-nums;
  transform: translate(-50%, 0);
  animation: t-pill-bob 2.6s ease-in-out infinite,
             t-pill-glow 2.2s ease-in-out infinite;
}
.t-nudge-cta-badge::before {                    /* upward arrow */
  content: ''; position: absolute;
  top: -4px; left: 50%;
  width: 8px; height: 8px; background: #ff5500;
  transform: translateX(-50%) rotate(45deg);
  border-radius: 1px;
}
```

CTA wrapper: `relative` + `mb-10` when `showNudge` (40px breathing room before feature list).

Animations:
- **`t-pill-bob`** 2.6s ∞ — vertical 0 → -3px → 0 (preserves `translateX(-50%)` centering)
- **`t-pill-glow`** 2.2s ∞ — `box-shadow: 0 0 0 0 rgba(255,85,0,0.55) → 0 0 0 9px rgba(255,85,0,0)` ripple ring expanding outward then dissolving

---

## 5. Feedback modal (close-intent)

Triggered by clicking the close X on the pricing modal (when `feedbackSubmitted=false`).

```
┌─────────────────────────────────────────┐    ╭─╮
│ ╭─ orange gradient header (180px) ──╮  │  X │ │  ← top-right, outside card
│ │ 🎁  SHARE FEEDBACK AND GET         │  │   ╰─╯
│ │     $60                            │  │
│ │     off Pro Annual                 │  │
│ ╰────────────────────────────────────╯  │
│                                          │
│  Your feedback                           │
│                                          │
│  ┌────────────────────────────────────┐ │
│  │  In just 2 minutes I built…        │ │
│  │                                    │ │
│  │  [😀 Happy] [⚡ Fast] [📝 …        │ │ ← marquee R→L
│  └────────────────────────────────────┘ │
│                                          │
│  [🔒 Submit feedback and unlock discount] │
│  We may share your testimonial for marketing. │
└─────────────────────────────────────────┘
```

### Header copy

| Element | Form step | Success step |
|---|---|---|
| **Tag (`.t-badge`)** | "Share Feedback and Get" → CSS uppercase: `SHARE FEEDBACK AND GET` | `THANK YOU` |
| **Amt** | EDU `50` (with `%` suffix) · Tier 1 `$60` | (same — number stays after reveal) |
| **Lbl** | "off Pro Annual" | "off Pro Annual Unlocked" |
| **Sub** | (none) | (none) |

### `$60` / `50%` reveal — V8 Slow Drift dial

**Two-stage animation matching 2-4s server response:**

1. **Hold (0 → 1200ms):** digits sit at `0`, blurred 6px (`[data-price-loading="true"]`)
2. **Spin (1200ms → ~5400ms):** blur clears + slot-machine slide
   - Tens digit: 5 full rotations + lands on target — 4.0s `cubic-bezier(0.16, 1, 0.3, 1)`
   - Ones digit: 6 full rotations + lands on `0` — 4.2s `cubic-bezier(0.16, 1, 0.3, 1)`
   - Filter `blur(6px) → blur(0)` over 3.6s/3.8s ease-out

**Markup:**
```jsx
<span className="t-amt" data-price-loading="true">
  <span className="t-amt-real">
    {!isEdu && <span className="t-sym">$</span>}
    <span className="t-digits">
      <span className="t-digit" data-target={isEdu ? "5" : "6"}><span className="t-digit-track"></span></span>
      <span className="t-digit" data-target="0"><span className="t-digit-track"></span></span>
    </span>
    {isEdu && <span style={{ fontSize: 22, ...}}>%</span>}
  </span>
</span>
```

**Track build (JS):** for each digit, append `cycles + 1` repetitions of `0..9`. Total height = `(cycles + 1) × 10 × 40px`. Slide offset = `(cycles × 10 + targetDigit) × 40px`.

**Edge fade mask** on `.t-digit`: `mask-image: linear-gradient(to bottom, transparent 0%, #000 30%, #000 70%, transparent 100%)` — digits visibly fade in from above + out to below as they slide through the 40px viewport.

```css
.t-digit { width: 26px; height: 40px; overflow: hidden; }
.t-digit-track > span { height: 40px; line-height: 40px; font-weight: 800; color: #ff5500; }
.t-digit-track { transition: transform 4s cubic-bezier(0.16, 1, 0.3, 1), filter 3.6s ease-out; }
.t-digit:nth-child(2) .t-digit-track { transition: transform 4.2s cubic-bezier(0.16, 1, 0.3, 1), filter 3.8s ease-out; }
.t-amt[data-price-loading="true"] .t-digit-track { filter: blur(6px); }
.t-amt[data-price-loading="false"] .t-digit-track { filter: blur(0); }
```

### Body — testimonial picker

8 emoji feelings × 5 quotes each = **40 randomized presentation testimonials**. Marquee scrolls right→left at 60s linear ∞. Doubled chips for seamless loop. Pause on row hover.

```js
const FEELINGS = [
  { em: "😀", label: "Happy",      qs: [...5 quotes] },
  { em: "⚡️", label: "Fast",        qs: [...] },
  { em: "📝", label: "Detailed",    qs: [...] },
  { em: "🤯", label: "Mind blown",  qs: [...] },
  { em: "😍", label: "Love it",     qs: [...] },
  { em: "😲", label: "Surprised",   qs: [...] },
  { em: "🙌", label: "Grateful",    qs: [...] },
  { em: "🖌️", label: "Crafted",     qs: [...] },
];
```

Click chip → random `qs[]` entry fills `.t-quote` (contenteditable). Card toggles `.picked` → marquee fades + slides 8px down. Submit CTA unlocks (lock-pop + cta-shine).

User can type own testimonial. CTA enables on first char, disables on backspace-to-empty.

### Form heading

```jsx
<h3 className="t-heading" style={{ fontSize: 16, fontWeight: 600, marginBottom: 12 }}>Your feedback</h3>
```

### Disclaimer

```jsx
<p className="t-disclaimer" style={{ textAlign: "center", marginTop: 12 }}>
  We may share your testimonial for marketing.
</p>
```

Position: **below** the Submit button, center-aligned, 12px gray.

### Submit / close handlers

Both routes apply the bumped state:

```js
const handleFeedbackSubmit = () => {
  setFeedbackSubmitted(true);
  setDiscountTier("bumped");
  setBumpStart(Date.now());
  setNow(Date.now());
  setFeedbackOpen(false);
};

const handleFeedbackClose = () => {
  // Same — close without submit ALSO applies discount
  setFeedbackSubmitted(true);
  setDiscountTier("bumped");
  setBumpStart(Date.now());
  setNow(Date.now());
  setFeedbackOpen(false);
};
```

### Success step

| Element | Content |
|---|---|
| Tag (`.t-badge`) | `THANK YOU` |
| Title | (no `<h2>` — tag is the title) |
| Body | "We heard you. As a token of appreciation, we've unlocked **$60 off Pro Annual** [or **50% off** for EDU] for the next **hour**." |
| CTA | "Buy Pro Annual at $60 off" |
| Floating timer | "Ends in MM:SS" pulsing badge above CTA (60-min countdown) |

---

## 6. CTA — lock-pop + shine (`.fb-btn-primary`)

```jsx
<button className="fb-btn-primary" disabled={submitDisabled}>
  <span className="fb-lock">
    <i className="ph-fill ph-lock"></i>      {/* visible while disabled */}
    <i className="ph-fill ph-lock-open"></i> {/* visible on enable */}
  </span>
  Submit feedback and unlock discount
</button>
```

- Background: `linear-gradient(180deg, #1c3550 0%, #0A1925 100%)`. Disabled: `#e5e5e5` / `#a3a3a3`.
- **`lock-pop`** keyframe (0.55s spring, `cubic-bezier(0.34, 1.35, 0.64, 1)`): `-18deg/0.6 → 8deg/1.15 → 0deg/1.0` — fires when `:not([disabled])` flips on first emoji pick or first typed character.
- **`cta-shine`** keyframe (0.75s linear): white sweep from `150% → -50%` on `.just-unlocked` class. Re-triggered each time CTA goes from disabled → enabled (forced reflow `void cta.offsetWidth`).

---

## 7. Animation cheat-sheet

| Keyframe / pattern | Duration | Where | Purpose |
|---|---|---|---|
| `gift-bounce` | 1.8s ∞ | `.gift-emoji` | 🎁 bobs up/down (-5px) |
| `t-conf-drift` | 4.6s ∞ varied | `.t-conf` confetti pieces | drift + rotate, 5 pieces in header |
| `ticker-scroll` | 40s linear ∞ | `.uni-ticker-track` | logos scroll right→left |
| `emoji-marquee` *(t-emoji-track)* | 60s linear ∞ | testimonial picker chips | right→left infinite scroll |
| `t-digit-track transition` | 4s / 4.2s ease-out | `.t-digit-track` | slot-machine spin (V8 slow drift), `cubic-bezier(0.16, 1, 0.3, 1)` |
| `t-digit-track filter` | 3.6s / 3.8s ease-out | same | blur(6px) → blur(0) reveal |
| `lock-pop` | 0.55s spring | `.fb-lock .ph-lock-open` | padlock unlocks on submit-enable |
| `cta-shine` | 0.75s linear | `.fb-btn-primary.just-unlocked::before` | white sweep across submit button |
| `fb-cta-badge-pulse` | 1.8s ∞ | `.fb-cta-badge` (success countdown) | vertical bob (-4px) |
| `fb-cta-badge-ripple` | 2.2s ∞ | same | ring `0 → 12px rgba(255,85,0,0.55→0)` ripple |
| `t-pill-bob` | 2.6s ∞ | `.t-nudge-cta-badge` (Pro nudge chip) | vertical bob (-3px) |
| `t-pill-glow` | 2.2s ∞ | same | ring `0 → 9px rgba(255,85,0,0.55→0)` ripple |
| `chip-icon-spin` | 4s linear ∞ | `.ph-fill ph-seal-percent` (accent feature row) | seal rotates 360° |

---

## 8. CSS tokens

| Color | Hex | Usage |
|---|---|---|
| Brand orange | `#ff5500` | Header digit fill, badge bg, nudge chip bg, accents |
| Hot orange | `#ff732d` | Confetti, secondary orange |
| Light orange | `#ff801a` | Confetti |
| Tag bg | `rgba(255,85,0,0.22)` | `.t-badge`, `.fb-tag` |
| Tag text | `#c64200` | Tag chip text |
| Header gradient | `#ffdcbf → #fff3e6` | `.t-nudge-header` 164.93deg |
| Ink primary | `#171717` | Strong text, "off Pro Annual" lbl |
| Ink body | `#525252` | Body text, feature labels |
| Ink mute | `#8d8d8d` | Disclaimer |
| Card bg (Pro) | `linear-gradient(125.62deg, #ffffff 2.19%, #ffffff 41.38%, #eef2f6 98.14%)` | Subtle gradient |
| Card border (Pro) | `#b8c1cc` 2px | Pro card stroke |
| Card border (Single) | `#e5e7eb` 1px | Single PPT outline |
| Most popular | `#ff5500` | Top-right pill on Pro card |
| CTA gradient | `#1c3550 → #0A1925` | `.fb-btn-primary`, `CardCta` (navy) |
| Outline CTA stroke | `#0A1925` | `OutlinedCta` border + text |
| Accent green | `#16a34a` | Discount-label feature row |
| Confetti palette | `#4285f4`, `#16a34a`, `#fbbc04`, `#f24000` | Drift dots |

| Sizing token | Value | Usage |
|---|---|---|
| Header height | 180px | `.t-nudge-header` |
| Modal width | 520px | `.t-offer-nudge` max-width |
| Title row min-height | 68px | Pro + Single PPT card title rows |
| Buy CTA height | 48px (h-12) | OutlinedCta + CardCta |
| Submit CTA height | 48px | `.fb-btn-primary` |
| Digit cell | 26 × 40px | `.t-digit` viewport |
| Digit row | 40px | `.t-digit-track > span` |

---

## 9. JS surface

### Page state
- `audience: "tier1" | "edu"` — toggle
- `discountTier: "base" | "bumped"` — Pro Plan pricing branch
- `feedbackOpen: bool` — feedback modal visibility
- `feedbackSubmitted: bool` — guards re-opening on close
- `bumpStart: number | null` — Date.now() when bumped started, drives countdown
- `now: number` — drives live countdown updates

### Effects
- Countdown ticker — `setInterval(() => setNow(Date.now()), 1000)` while `feedbackOpen` or `bumpStart !== null`
- Auto-revert — when `now - bumpStart >= BUMP_DURATION_MS (60min)` → `setDiscountTier("base")`

### FeedbackModal internals
- `feeling`, `quote`, `step` state
- `submitBtnRef`, `quoteRef`, `wasDisabledRef` refs
- `useEffect` on `open`: reset feeling/quote/step + clear quoteRef text
- `useEffect` on `open`: build digit tracks (cycles 5/6) + 1.2s hold + slide to target
- `useEffect` on `submitDisabled`: lock-pop + shine fire when CTA flips disabled → enabled

---

## 10. Logos strip

Bottom of card carousel — audience-aware:

| Audience | Logos | Source |
|---|---|---|
| Tier 1 | Adobe, EY, BCG, Amazon, Facebook, Google, McKinsey, Microsoft, Notion (9) | `assets/export-ppt/logos/*.svg` |
| EDU | logo1-8.png — universities (8) | `assets/export-ppt/logos-edu/*.png` |

Logos: 40px tall, padding `0 18px`, grayscale 100%, opacity 0.6 (no hover). Doubled for seamless marquee. `ticker-scroll 40s linear ∞`.

---

## 11. Testimonial panel (left side of pricing modal)

```
┌─────────────────────────────╮
│ "Quote from testimonial      │
│  here…" (Hedvig Letters     │
│  Serif italic, 24px)         │
│                              │
│         ┌────────┐ Marcus C. │
│         │portrait│ VP Strat. │
│         │        │ Pro Annual│
│         └────────┘ ★★★★★      │
└─────────────────────────────╯
```

| Audience | Portrait | Person |
|---|---|---|
| Tier 1 | `assets/export-ppt/illo2.png` | Marcus Chen — VP Strategy · Fortune 500 — Pro Annual member |
| EDU | `assets/export-ppt/priya.png` (flipped via `transform: scaleX(-1)`) | Jenny Wong — Stanford GSB — Pro member |

Image: 150×150 `object-contain` with `mixBlendMode: multiply`, `-ml-6` bleed.
Bio: `py-5 pr-5` (no left padding, `-ml-2`), `space-y-1`, name 14px/600, school+plan 12px gray-500, stars 12px orange.

---

## 12. Production checklist

1. **Strip the audience toggle** — 3-tab debug switcher. Pick one mode based on user segmentation (auth state, GrowthBook flag, A/B id).
2. **Wire submit to backend** — `handleFeedbackSubmit` only updates local state. POST `{ feeling, quote }` to `/api/feedback`.
3. **Persist discount across refresh** — `bumpStart` is in-memory only. Store in `localStorage` keyed by user id so refresh doesn't lose the bumped offer.
4. **Wire claim CTA** — "Buy Pro Annual at $60 off" only flips local `discountTier`. Should route to checkout with `?promo=PROANNUAL60` (or EDU equivalent).
5. **Strip Agentation block** — dev-only widget at line ~1750. Do not ship.
6. **Replace skeleton hold-timer with real fetch** — currently `setTimeout(1200)`. Replace with actual API resolve. Flip `data-price-loading="false"` on resolve + trigger digit slide.
7. **Audit unused constants** — `ONE_TIME_BUMPED`, `ONE_TIME_TIER1_BUMPED` no longer rendered (Single PPT never bumps). Either delete or keep for future re-enable.
8. **A11y audit:**
   - Phosphor icons need `aria-hidden="true"`
   - `.t-emoji-chip` chips: add `aria-label` (currently `title` only)
   - Focus order: quote → submit
   - Reduce-motion: add `@media (prefers-reduced-motion: reduce)` to disable digit spin + cta-shine + pill ripple
9. **Mobile breakpoint** — modal currently fixed-width 1200px max. Add stacking media query for <768px (cards stack vertically, testimonial panel collapses or moves below).
10. **Localize all visible copy** — including FEELINGS quotes, button labels, success body.

---

## 13. File asset map

```
assets/export-ppt/
├─ illo2.png            # Tier 1 testimonial portrait (Marcus Chen)
├─ illo5.png            # alt EDU portrait (unused — priya.png used instead)
├─ priya.png            # EDU testimonial portrait (Jenny Wong, flipped)
├─ icons-v2.png         # Single PPT card illustration (PowerPoint stack)
├─ logos/               # Tier 1 brand logos (9 SVGs)
│  └─ adobe / amazon / bcg / ey / facebook / google / mckinsey / microsoft / notion
└─ logos-edu/           # EDU university logos (8 PNGs)
   └─ logo1.png … logo8.png
```

Pro Plan card uses inline base64 PNG for `.illus-pro` (rocket illus). Single PPT uses `assets/export-ppt/icons-v2.png` via `.illus-basic`.

---

## 14. Versioning

| Date | Change |
|---|---|
| 2026-04-29 | Initial scaffold. Modal-over-dashboard layout. Variation A pricing modal. EDU + Tier 1 audience toggle. |
| 2026-04-30 | FeedbackModal redesign — testimonial picker (text field + emoji marquee) replaces survey. Lock-pop CTA. Audience-aware testimonials. |
| 2026-05-04 | Pricing reset: Tier 1 base $119/$20, bumped $49/$15. EDU base $59/$15, bumped $59/$10. Tier 1 BUMPED + "offer seen" tab. Single PPT never discounts. Logos audience-aware (EDU universities vs Tier 1 brands). NudgeChip pulse + ripple ring. Skeleton + scale-pop reveal on $60. EDU header amt = "50%". Submit copy "Submit feedback and unlock discount". Feature icons: `ph-fill ph-seal-percent` (discount) + `ph ph-pencil-simple` (editable export). Title row + button alignment unified across cards (`min-h-[68px]`). Pro Plan label strips ` • XX% Off` when nudge active. Close X applies discount same as submit. |
| 2026-05-04 | Body heading "Pick a feeling - we'll write the testimonial" → **"Your feedback"** (16px/600). Disclaimer relocated **below** the Submit button + center-aligned + shortened to "We may share your testimonial for marketing." |
| 2026-05-04 | **V8 slow-drift digit dial replaces skeleton.** 2-stage: 1.2s blurred hold → 4.0s/4.2s silky multi-rotation slide via `cubic-bezier(0.16, 1, 0.3, 1)`. Edge fade mask top/bottom. Header height `140 → 180px`. Tag copy "Help us improve and get" → "Share Feedback and Get" / "Feedback received" → "Thank you". Success body "for the next one hour" → "for the next hour". Lbl "off Pro Annual" → "off Pro Annual Unlocked" on success. Toggle labels capitalized. Single PPT button unified to "Export a single PPT". Pro Plan Tier 1 button "Buy Now" → "Export unlimited PPT with Pro". Pro feature row "Unlimited exports" → "Export unlimited decks". |
| 2026-05-04 | **Mobile responsive (md = 768px breakpoint).** Outer grid `grid-cols-1 md:grid-cols-[35fr_65fr]`. Mobile stack order: Pro card → Single PPT card → testimonial panel → logos. Testimonial panel `order-2 md:order-1` (bottom mobile, left desktop). Cards inner grid `grid-cols-1 md:grid-cols-2`. Card order: Pro `order-1`, Single `order-2` on mobile (Pro first); desktop natural left-right. Section padding `p-6 md:p-10`. Outer modal padding `py-6 px-3 md:py-16 md:px-6`. Close button moves inside modal on mobile (`top-2 right-2 md:top-0 md:-right-14`). **Logo strip fixed-position at viewport bottom on mobile** (`position: fixed; bottom: 0; z-40`) — content scrolls behind it. Modal shell reserves `padding-bottom: 80px` so last feature row clears the pinned strip. |
