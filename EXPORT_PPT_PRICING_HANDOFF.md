# Export PPT Pricing Modal — Engineering Handoff

**File:** `export-ppt-pricing.html` (~1700 lines, single self-contained HTML)

**Purpose:** Modal-over-dashboard pricing screen. User finishes a deck → sees this modal with two cards (Single PPT one-time + Pro Plan annual) and a testimonial panel. Close-intent triggers a feedback modal that, on submit OR close, applies a deeper discount to the Pro Plan card.

Three audience modes (top toggle, debug only — pick one for production):

| Mode | Pricing | Notes |
|---|---|---|
| **Tier 1 — offer not seen** | Single PPT $119, Pro Plan $20/mo | Default state. No nudge chip. |
| **Tier 1 — offer seen** | Single PPT $119, Pro Plan **$15/mo** (strikeout $20) | Bumped state without close-feedback flow. Pulsing chip + countdown active. |
| **EDU** | Single PPT $59, Pro Plan **$15/mo** (strikeout $20, 25% education discount) | Default. Close X triggers feedback modal → bumped to **$10/mo** (50% student discount). |

---

## 1. Tech stack

- React 18 via esm.sh (no build step, importmap)
- Babel standalone for in-browser JSX compile
- Tailwind Play CDN
- Phosphor Icons (regular + fill + bold) via CDN
- Inter (400/500/600/700/800) + Hedvig Letters Serif + Instrument Serif via Google Fonts
- Agentation@3 dev-only feedback widget (mounted via esm.sh module script)

No bundler. Open file directly in any static server.

---

## 2. Pricing data model

### Constants (top of `<script>`)

```js
const ONE_TIME_BASE = {           // EDU Single PPT — always shown
  shortName: "Single PPT",
  originalPrice: 59, price: 59,
  hideStrikeout: true,             // never discounted
  discountLabel: null,
  buyLabel: "Get a single PPT",
  compareDiscount: "Single export",
};

const ONE_TIME_BUMPED = {          // unused — Single PPT never bumps
  price: 9, discountLabel: "90% one-time discount", ...
};

const UNLIMITED_BASE = {           // EDU Pro Plan — base
  shortName: "Pro Plan",
  originalPriceMonthly: 20,
  studentPriceMonthly: 15,         // 25% off
  discountLabel: "25% education discount",
  buyLabel: "Buy Now • 25% Off",
  compareDiscount: "25% education discount",
  billed: "billed annually",
};

const UNLIMITED_BUMPED = {         // EDU Pro Plan — after close-feedback
  studentPriceMonthly: 10,         // 50% off
  discountLabel: "50% student discount",
  buyLabel: "Buy Now • 50% Off",
};

const ONE_TIME_TIER1_BASE = {      // Tier 1 Single PPT
  originalPrice: 119, price: 119,
  hideStrikeout: true,
  discountLabel: null,
  buyLabel: "Get a single PPT",
};

const UNLIMITED_TIER1_BASE = {     // Tier 1 Pro Plan — base
  originalPriceMonthly: 20,
  studentPriceMonthly: 20,
  hideStrikeout: true,
  discountLabel: null,
  buyLabel: "Buy Now",
  billed: "billed annually",
};

const ONE_TIME_TIER1_BUMPED = {    // unused — Single PPT never bumps
  price: 49, discountLabel: "$70 off Single PPT", ...
};

const UNLIMITED_TIER1_BUMPED = {   // Tier 1 Pro Plan — after offer
  studentPriceMonthly: 15,         // $5/mo off → $60/yr
  discountLabel: "$60 off Pro Annual",
  buyLabel: "Buy Now",
};
```

### Derivation in `Page()`

```js
// Single PPT NEVER discounts — always base across all tabs
const oneTime = audience === "tier1" ? ONE_TIME_TIER1_BASE : ONE_TIME_BASE;

// Pro Plan flips between BASE / BUMPED based on discountTier
const unlimited = audience === "tier1"
  ? (discountTier === "bumped" ? UNLIMITED_TIER1_BUMPED : UNLIMITED_TIER1_BASE)
  : (discountTier === "bumped" ? UNLIMITED_BUMPED : UNLIMITED_BASE);

const testimonial = audience === "tier1" ? TESTIMONIAL_TIER1 : TESTIMONIAL;

const extraOneTime = audience === "tier1"
  ? ONE_TIME_TIER1_BASE.price - ONE_TIME_TIER1_BUMPED.price          // 119 - 49 = 70 (unused)
  : ONE_TIME_BASE.price - ONE_TIME_BUMPED.price;                      // 49 - 9 = 40 (unused)

const extraUnlimited = audience === "tier1"
  ? (UNLIMITED_TIER1_BASE.studentPriceMonthly - UNLIMITED_TIER1_BUMPED.studentPriceMonthly) * 12  // 5 × 12 = 60 annual
  : (UNLIMITED_BASE.studentPriceMonthly - UNLIMITED_BUMPED.studentPriceMonthly) * 12;             // 5 × 12 = 60 annual
```

### Audience toggle (top-center fixed pill)

```jsx
{[
  { key: "tier1",     label: "offer not seen" },
  { key: "tier1seen", label: "offer seen"     },
  { key: "edu",       label: "EDU"            },
].map(...)
```

`tier1seen` click handler:
```js
setAudience("tier1");
setDiscountTier("bumped");
setBumpStart(Date.now());
setNow(Date.now());
setFeedbackSubmitted(true);    // suppresses re-opening feedback modal
setFeedbackOpen(false);
```

Strip the toggle for production. Pick one mode based on user segmentation (auth state, query param, A/B flag).

---

## 3. Card layout

```
┌─────────────────────────────────────────────┐
│  ╭──────────────────╮  ╭─────────────────╮  │
│  │ illus            │  │ 🚀 illus  [most │  │
│  │ ┌──────┬──────┐  │  │           popular]│  │
│  │ │Single│  $59 │  │  │ Pro Plan  $15/mo │  │
│  │ │ PPT  │billed│  │  │           $20 ↗  │  │
│  │ │      │ once │  │  │           billed │  │
│  │ └──────┴──────┘  │  │           annually│  │
│  │ [Get a single PPT]│ │ [Buy Now • 25% Off]│ │
│  │                  │  │ ┌──── nudge chip─┐│ │
│  │ ✏️ Export this   │  │ │ ⚡ 50% off · │  │ │
│  │ ✏️ Pixel-perfect │  │ │ Ends in 59:51│  │ │
│  │                  │  │ └──────────────┘  │ │
│  │                  │  │ 🔖 25% education  │ │
│  │                  │  │ ✏️ Unlimited      │ │
│  │                  │  │ ✏️ Pixel-perfect  │ │
│  │                  │  │ 🪙 5,000 Credits  │ │
│  │                  │  │ ⭐ Advanced AI    │ │
│  ╰──────────────────╯  ╰─────────────────╯  │
└─────────────────────────────────────────────┘
```

### Title row alignment

Both cards: `min-h-[68px]` reserved on the title-row container + price column. Pro's 3-line price (strikeout / price/mo / billed) and Single's 2-line price ($X / billed once) both occupy the same vertical space → titles + buttons align across cards.

### Buttons

- Both `h-12 rounded text-[14px] font-bold` — unified size
- `OutlinedCta` (Single PPT): white bg, dark `#0A1925` border, dark text
- `CardCta` (Pro Plan): navy gradient `linear-gradient(180deg, #1c3550 0%, #0A1925 100%)`, white text
- Pro Plan button label: when `showNudge=true`, regex strips ` • XX% Off` suffix → label becomes "Buy Now" (chip carries the discount info)

### Feature list

```js
buildOneTimeFeatures = [
  { icon: "ph-fill ph-seal-percent",  text: oneTime.discountLabel,           accent: true },  // hidden if null
  { icon: "ph ph-file-ppt",           text: "Export just this deck" },
  { icon: "ph ph-pencil-simple",      text: "Pixel-perfect, fully editable export" },
].filter((f) => f.text);

buildUnlimitedFeatures = [
  { icon: "ph-fill ph-seal-percent",  text: unlimited.discountLabel,         accent: true },
  { icon: "ph ph-file-ppt",           text: "Unlimited exports" },
  { icon: "ph ph-pencil-simple",      text: "Pixel-perfect, fully editable export" },
  { icon: "ph ph-coin",               text: "5,000 Credits" },
  { icon: "ph ph-star-four",          text: "Advanced AI models and agents" },
].filter((f) => f.text);
```

Accent rows: green text `#16a34a`, spinning `ph-fill ph-seal-percent`. Other rows: gray `#525252`.

---

## 4. Nudge chip (`.t-nudge-cta-badge`)

Floats below Pro Plan Buy button when `discountTier === "bumped"`. Centered, arrow up, orange.

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
  padding: 5px 12px; background: #ff5500; color: #fff;
  border-radius: 999px;
  font-size: 11px; font-weight: 800; letter-spacing: 0.06em;
  font-variant-numeric: tabular-nums;
  transform: translate(-50%, 0);
  animation: t-pill-bob 2.6s ease-in-out infinite,
             t-pill-glow 2.2s ease-in-out infinite;
}
.t-nudge-cta-badge::before {                    /* upward arrow */
  content: ''; position: absolute; top: -4px; left: 50%;
  width: 8px; height: 8px; background: #ff5500;
  transform: translateX(-50%) rotate(45deg);
  border-radius: 1px;
}
```

Parent CTA wrapper: `relative` + `mb-10` when `showNudge` true (40px breathing room before feature list).

---

## 5. Feedback modal (close-intent)

Triggered by clicking the close X on the pricing modal (when `feedbackSubmitted=false`).

```
┌─────────────────────────────────────────┐    ╭─╮
│ ╭─ orange gradient header (140px) ──╮  │  X │ │  (top-right, outside)
│ │ 🎁  HELP US IMPROVE…               │  │   ╰─╯
│ │     $60 off Pro Annual             │  │
│ ╰────────────────────────────────────╯  │
│                                          │
│  Pick a feeling - we'll write the       │
│  testimonial                             │
│                                          │
│  ┌────────────────────────────────────┐ │
│  │  In just 2 minutes I built…        │ │
│  │                                    │ │
│  │  [😀 Happy] [⚡ Fast] [📝 Detail.. │ │ ← marquee R→L
│  └────────────────────────────────────┘ │
│                                          │
│  We may share your testimonial...        │
│                                          │
│  [🔒 Submit feedback and unlock discount]│
└─────────────────────────────────────────┘
```

### Header (audience-aware)

| Element | EDU | Tier 1 |
|---|---|---|
| Tag | "Share feedback, get" → "Feedback received" on submit | same |
| Amt | **`50%`** (number 50 + small `%` suffix) | **`$60`** (with `$` superscript) |
| Lbl | "off Pro Annual" | "off Pro Annual" |

Skeleton load → 2s peach shimmer placeholder → reveal via `t-price-pop` 0.5s spring (scale 0.85 → 1.06 overshoot → 1.0). No counting/odometer.

### Body — testimonial picker

8 emoji feelings × 5 quotes each = 40 randomized presentation testimonials. Marquee scrolls right→left at 60s linear infinite. Doubled chips for seamless loop. Pause on row hover.

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

User can also type own testimonial. CTA enables on first char, disables on backspace-to-empty.

### Submit / close handlers

Both routes apply the bumped state:

```js
const handleFeedbackSubmit = () => {
  setFeedbackSubmitted(true);
  setDiscountTier("bumped");        // → Pro Plan flips to bumped pricing
  setBumpStart(Date.now());          // → starts 60-min countdown
  setNow(Date.now());
  setFeedbackOpen(false);            // form swaps to success step
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

| Element | EDU | Tier 1 |
|---|---|---|
| Tag | "Feedback received" | "Feedback received" |
| Title | "Thank you" *(Instrument Serif italic 42px)* | same |
| Body strong | "50% off" | "$60 off Pro Annual" |
| CTA | "Buy Pro Annual at $60 off" | same |

Pulsing orange `.fb-cta-badge` floats above CTA: "Ends in MM:SS" with bob + ripple ring (3 concurrent animations).

---

## 6. Animation cheat-sheet

| Keyframe | Duration | Where | Purpose |
|---|---|---|---|
| `gift-bounce` *(testimonial.html)* / unused here | — | — | — |
| `t-conf-drift` | 4.6s ∞ | `.t-conf` confetti pieces | Drift + rotate |
| `t-emoji-marquee` | 60s linear ∞ | `.t-emoji-track` | Right→left infinite scroll |
| `t-skeleton-shimmer` | 1.1s ∞ | `.t-amt-skeleton` | Peach gradient sweep during 2s load |
| `t-price-pop` | 0.5s spring | `.t-amt-real` on reveal | Scale 0.85 → 1.06 → 1.0 + fade |
| `t-pill-bob` | 2.6s ∞ | `.t-nudge-cta-badge` (Pro nudge chip) | Vertical bob |
| `t-pill-glow` | 2.2s ∞ | same | Ripple ring expanding 0→9px |
| `lock-pop` | 0.55s spring | `.fb-lock .ph-lock-open` | Padlock unlocks on submit-enable |
| `cta-shine` | 0.75s linear | `.fb-btn-primary.just-unlocked::before` | White sweep across submit |
| `fb-cta-badge-pulse` | 1.8s ∞ | `.fb-cta-badge` (success countdown) | Vertical bob |
| `fb-cta-badge-ripple` | 2.2s ∞ | same | Ring expanding 0→12px |
| `fb-arrow-pulse` | 1.4s ∞ | `.fb-panel-sub .arrow` *(unused — sub line removed)* | — |
| `chip-shimmer` | inherited from base CSS | shimmer on chips | — |

---

## 7. CSS tokens

| Color | Hex | Usage |
|---|---|---|
| Brand orange | `#ff5500` | Gradient stops, accents, badge bg |
| Hot orange | `#ff732d` | Gradient mid, confetti |
| Tag bg | `rgba(255,85,0,0.22)` | `.t-badge` |
| Tag text | `#c64200` | `.t-badge` color |
| Header gradient | `#ffdcbf → #fff3e6` | `.t-nudge-header` 164.93deg |
| Ink primary | `#171717` | Strong text |
| Ink body | `#525252` | Body text |
| Card bg (Pro) | `linear-gradient(125.62deg, #ffffff 2.19%, #ffffff 41.38%, #eef2f6 98.14%)` | Pro card subtle gradient |
| Card border (Pro) | `#b8c1cc` 2px | Pro card stroke |
| Card border (Single) | `#e5e7eb` 1px | Single PPT outline |
| Most popular | `#ff5500` | Top-right pill |
| CTA gradient | `#1c3550 → #0A1925` | `.fb-btn-primary` (navy) |
| Outline CTA stroke | `#0A1925` | `.OutlinedCta` border + text |
| Accent green | `#16a34a` | Discount-label feature row |
| Confetti palette | `#4285f4`, `#16a34a`, `#fbbc04`, `#f24000`, `#ff801a` | Drift dots |

---

## 8. JS surface

### Page state
- `audience: "tier1" | "edu"` — toggle
- `discountTier: "base" | "bumped"` — pricing branch for Pro Plan
- `feedbackOpen: bool` — feedback modal visibility
- `feedbackSubmitted: bool` — guards re-opening on close
- `bumpStart: number | null` — Date.now() when bumped started, drives countdown
- `now: number` — used for live countdown updates

### Effects
- Countdown ticker — runs `setInterval(setNow(Date.now()), 1000)` while `feedbackOpen` or `bumpStart !== null`
- Auto-revert — when `now - bumpStart >= BUMP_DURATION_MS (60min)` → `setDiscountTier("base")`

### FeedbackModal internals
- `feeling, quote, step` state
- `submitBtnRef, quoteRef, wasDisabledRef` refs
- `useEffect` on `open`: reset feeling/quote/step + clear quoteRef text
- `useEffect` on `open`: 2s skeleton timer + reset `data-price-loading="true"` for next open
- `useEffect` on `submitDisabled`: lock-pop + shine fire when CTA flips disabled → enabled

---

## 9. Logos strip

Bottom of card carousel — audience-aware:

| Audience | Logos | Source |
|---|---|---|
| Tier 1 | Adobe, EY, BCG, Amazon, Facebook, Google, McKinsey, Microsoft, Notion (9) | `assets/export-ppt/logos/*.svg` |
| EDU | logo1-8.png — universities (8) | `assets/export-ppt/logos-edu/*.png` |

Logos: 40px tall, padding `0 18px`, grayscale 100%, opacity 0.6 (no hover state). Doubled for seamless marquee. `ticker-scroll 40s linear infinite`.

---

## 10. Testimonial panel (left side of pricing modal)

```
┌─────────────────────────────╮
│ "Quote from testimonial      │
│  here…" *(Hedvig Letters     │
│  Serif italic, 24px)*        │
│                              │
│         ┌───┐  Marcus Chen   │
│         │ 👤│  VP Strategy   │
│         │   │  Pro Annual…   │
│         └───┘  ★★★★★         │
└─────────────────────────────╯
```

| Audience | Portrait | Person |
|---|---|---|
| Tier 1 | `assets/export-ppt/illo2.png` | Marcus Chen — VP Strategy · Fortune 500 — Pro Annual member |
| EDU | `assets/export-ppt/priya.png` (flipped via scaleX(-1)) | Jenny Wong — Stanford GSB — Pro member |

Image: 150×150 `object-contain` with `mixBlendMode: multiply`, `-ml-6` bleed.
Bio: `py-5 pr-5` (no left padding, `-ml-2`), `space-y-1`, name 14px/600, school+plan 12px gray-500, stars 12px orange.

---

## 11. Production checklist

1. **Strip the audience toggle** — 3-tab switcher is debug only. Pick one mode based on user segmentation (auth state, GrowthBook flag, A/B id).
2. **Wire submit to backend** — `handleFeedbackSubmit` currently only updates local state. POST to `/api/feedback` with `{ feeling, quote }`.
3. **Persist discount across refresh** — `bumpStart` is in-memory only. Store in `localStorage` keyed by user id so refresh doesn't lose the bumped offer.
4. **Wire claim CTA** — "Buy Pro Annual at $60 off" currently only flips local `discountTier`. Should route to checkout with `?promo=PROANNUAL60` (or EDU equivalent).
5. **Strip Agentation block** — dev-only widget at line ~1750. Do not ship.
6. **Strip skeleton timer if API gate is real** — replace `setTimeout(2000)` with actual fetch promise; flip `data-price-loading="false"` on resolve.
7. **Audit unused constants** — `ONE_TIME_BUMPED`, `ONE_TIME_TIER1_BUMPED` no longer rendered (Single PPT never bumps). Either delete or keep for future re-enable.
8. **A11y audit:**
   - Phosphor icons need `aria-hidden="true"`
   - `.t-emoji-chip` chips: add `aria-label` (currently `title` only)
   - Focus order: quote → submit
9. **Mobile breakpoint** — modal currently fixed-width 1200px max. Add stacking media query for <768px (cards stack vertically, testimonial panel collapses or moves below).
10. **Localize all visible copy** — including FEELINGS quotes, button labels, success body.

---

## 12. File asset map

```
assets/export-ppt/
├─ illo2.png            # Tier 1 testimonial portrait (Marcus Chen)
├─ illo5.png            # alt EDU portrait (unused — priya.png used instead)
├─ priya.png            # EDU testimonial portrait (Jenny Wong, flipped)
├─ icons-v2.png         # Single PPT card illustration (PowerPoint stack)
├─ logos/               # Tier 1 brand logos (9 SVGs)
│  ├─ adobelogo.svg
│  ├─ amazonlogo.svg
│  ├─ bcglogo.svg
│  ├─ eylogo.svg
│  ├─ facebooklogo.svg
│  ├─ googlelogo.svg
│  ├─ mckinseylogo.svg
│  ├─ microsoftlogo.svg
│  └─ notionlogo.svg
└─ logos-edu/           # EDU university logos (8 PNGs)
   └─ logo1.png … logo8.png
```

Pro Plan card uses inline base64 PNG for `.illus-pro` (rocket illus). Single PPT uses `assets/export-ppt/icons-v2.png` via `.illus-basic`.

---

## 13. Versioning

| Date | Change |
|---|---|
| 2026-04-29 | Initial scaffold. Modal-over-dashboard layout. Variation A pricing modal. EDU + Tier 1 audience toggle. |
| 2026-04-30 | FeedbackModal redesign — testimonial picker (text field + emoji marquee) replaces survey. Lock-pop CTA. Audience-aware testimonials. |
| 2026-05-04 | Pricing reset: Tier 1 base $119/$20, bumped $49/$15. EDU base $59/$15, bumped $59/$10. Tier 1 BUMPED + "offer seen" tab. Single PPT never discounts. Logos audience-aware (EDU universities vs Tier 1 brands). NudgeChip pulse + ripple ring matching v2.html. Skeleton + scale-pop reveal on $60. EDU header amt = "50%". Submit copy "Submit feedback and unlock discount". Feature list icons: `ph-fill ph-seal-percent` (discount) + `ph ph-pencil-simple` (editable export). Title row + button alignment unified across cards (`min-h-[68px]`). Pro Plan label strips ` • XX% Off` when nudge active. Close X applies discount same as submit. |
| 2026-05-04 | Body heading "Pick a feeling - we'll write the testimonial" → **"Your feedback"** (16px/600). Disclaimer relocated **below** the Submit button + center-aligned + shortened to "We may share your testimonial for marketing." |
