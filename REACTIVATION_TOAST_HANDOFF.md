# Reactivation Toast — Engineering Handoff

**File:** `v2.html` (entry route: `/v2.html?bumped=1&toast=reactivated`)
**Live:** https://manivasakan-arch.github.io/pricing-20-4-2026/v2.html?bumped=1&toast=reactivated

A non-blocking confirmation toast that fires when a user lands on the pricing page after **reactivating** a previously expired discount. Slides from the **center-bottom**, drains over 6 seconds via a bottom progress bar, and dismisses itself (or on user click).

---

## 1. Trigger flow

```
Dashboard (header in EXPIRED state)
  └─ user clicks "Reactivate and Upgrade"
     └─ flips header → ACTIVE  (timer pill returns)
     └─ 350ms delay (lets pill restore visually)
     └─ redirect → v2.html?bumped=1&toast=reactivated
        │
        v
        v2.html on load reads query params:
          • bumped=1 → calls window.applyProBump()
                        — flips Pro card to bumped pricing
                        — sets localStorage.proBumpStart = Date.now()
                        — starts 60-min countdown chip
          • toast=reactivated → 400ms after layout settles
                                  → toast.dataset.open = 'true'
                                  → slides in + drain animation starts
                                  → auto-dismisses at 6s (or on X click)
```

### URL parameters

| Param | Value | Effect |
|---|---|---|
| `bumped` | `1` | Applies the deeper $60-off Pro Annual pricing on the page (calls `window.applyProBump()`) + starts the 60-min countdown |
| `toast` | `reactivated` | Triggers the reactivation toast 400ms after page paint |

Both are read independently — `?bumped=1` alone applies pricing without the toast; `?toast=reactivated` alone shows the toast without bumping pricing. Use both together for the full reactivation flow.

---

## 2. Markup

```html
<div class="reactivate-toast" id="reactivate-toast" data-open="false" role="status" aria-live="polite">
  <span class="icon"><i class="ph-fill ph-check-circle"></i></span>
  <div class="body">
    <h4>Reactivated discount for you, just this once</h4>
  </div>
  <button class="close-x" id="reactivate-toast-close" aria-label="Dismiss">
    <i class="ph ph-x"></i>
  </button>
  <div class="progress" aria-hidden="true"></div>
</div>
```

**Anatomy:**

| Slot | Purpose |
|---|---|
| `.icon` | Leading status icon (Phosphor `ph-fill ph-check-circle`, 20px, green `#16a34a`) |
| `.body > h4` | Single-line message, 14px Inter Medium, white |
| `.close-x` | Right-side dismiss button (16px Phosphor X, 70% white) |
| `.progress` | Bottom drain bar (3px) — track + scaleX-animated fill |

**`data-open="true|false"`** drives both the entrance transition AND the drain animation start.

---

## 3. CSS — fully customizable via custom properties

```css
.reactivate-toast {
  --toast-duration: 6s;                         /* lifespan + drain duration */
  --toast-bg: #0A1925;                          /* surface */
  --toast-track-bg: rgba(255,255,255,0.10);     /* progress track */
  --toast-fill-bg:  rgba(255,255,255,0.55);     /* progress fill */
  ...
}
```

Override per-instance to swap themes. Example light theme:

```css
.reactivate-toast.light {
  --toast-bg: #fff;
  --toast-track-bg: rgba(0,0,0,0.06);
  --toast-fill-bg:  #ff5500;
  color: #171717;
}
```

### Full block

```css
.reactivate-toast {
  /* Custom properties (override per-instance) */
  --toast-duration: 6s;
  --toast-bg: #0A1925;
  --toast-track-bg: rgba(255,255,255,0.10);
  --toast-fill-bg: rgba(255,255,255,0.55);

  position: fixed;
  bottom: 24px; left: 50%;
  background: var(--toast-bg);
  color: #fff;
  border-radius: 4px;
  padding: 12px;
  width: auto;
  max-width: calc(100vw - 32px);
  display: inline-flex; align-items: center; gap: 12px;
  white-space: nowrap;
  transform: translate(-50%, 32px);
  opacity: 0;
  pointer-events: none;
  transition:
    transform .35s cubic-bezier(0.16, 1, 0.3, 1),
    opacity   .25s ease;
  box-shadow: 0 12px 32px rgba(10,25,37,0.32);
  z-index: 200;
}
.reactivate-toast[data-open="true"] {
  transform: translate(-50%, 0);
  opacity: 1;
  pointer-events: auto;
}

.reactivate-toast .icon {
  width: 20px; height: 20px;
  color: #16a34a;
  display: inline-flex; align-items: center; justify-content: center;
  flex-shrink: 0;
}
.reactivate-toast .icon i { font-size: 20px; }

.reactivate-toast .body { flex: 1; min-width: 0; }
.reactivate-toast h4 {
  font-size: 14px; font-weight: 500;
  color: #fff; line-height: 1.43; margin: 0;
}

.reactivate-toast .close-x {
  width: 16px; height: 16px;
  color: rgba(255,255,255,0.7);
  background: transparent; border: 0;
  cursor: pointer; padding: 0;
  display: inline-flex; align-items: center; justify-content: center;
  flex-shrink: 0;
}
.reactivate-toast .close-x:hover { color: #fff; }
.reactivate-toast .close-x i { font-size: 16px; }

/* Bottom progress bar */
.reactivate-toast .progress {
  position: absolute;
  left: 0; right: 0; bottom: 0;
  height: 3px;
  background: var(--toast-track-bg);
  overflow: hidden;
  border-bottom-left-radius: 4px;
  border-bottom-right-radius: 4px;
}
.reactivate-toast .progress::after {
  content: '';
  position: absolute;
  left: 0; top: 0; bottom: 0;
  width: 100%;
  background: var(--toast-fill-bg);
  transform-origin: left center;
  transform: scaleX(1);
  will-change: transform;
}
.reactivate-toast[data-open="true"] .progress::after {
  animation: toast-drain var(--toast-duration) linear forwards;
}
.reactivate-toast:hover .progress::after {
  animation-play-state: paused;
}
@keyframes toast-drain {
  from { transform: scaleX(1); }
  to   { transform: scaleX(0); }
}
```

---

## 4. JS — init script

Reads URL params on page load, applies pricing bump, surfaces toast. Self-contained IIFE.

```js
(function () {
  const params = new URLSearchParams(window.location.search);

  // Apply Pro bump if requested via query string
  if (params.get('bumped') === '1') {
    if (typeof window.applyProBump === 'function') {
      window.applyProBump();
    } else {
      // applyProBump defined later in DOM — run after window load
      window.addEventListener('load', () => {
        if (typeof window.applyProBump === 'function') window.applyProBump();
      });
    }
  }

  // Surface reactivation toast
  if (params.get('toast') === 'reactivated') {
    const toast = document.getElementById('reactivate-toast');
    const closeBtn = document.getElementById('reactivate-toast-close');
    let autoTimer = null;

    const show = () => {
      if (!toast) return;
      toast.dataset.open = 'true';
      // Auto-dismiss MUST match --toast-duration (CSS drain animation)
      autoTimer = setTimeout(() => { toast.dataset.open = 'false'; }, 6000);
    };
    const hide = () => {
      if (!toast) return;
      toast.dataset.open = 'false';
      if (autoTimer) clearTimeout(autoTimer);
    };
    if (closeBtn) closeBtn.addEventListener('click', hide);

    // Wait for layout to settle so the slide-in is visible
    setTimeout(show, 400);
  }
})();
```

**Important:** the JS `setTimeout(6000)` MUST match the CSS `--toast-duration`. If you change one, change both. To make this single-source, you can read the computed property:

```js
const duration = parseFloat(
  getComputedStyle(toast).getPropertyValue('--toast-duration')
) * 1000;
autoTimer = setTimeout(() => { toast.dataset.open = 'false'; }, duration);
```

---

## 5. Animation breakdown

| Phase | Duration | Mechanism |
|---|---|---|
| **Idle** (`data-open="false"`) | — | `transform: translate(-50%, 32px); opacity: 0; pointer-events: none` — offscreen below |
| **Slide in** | 0.35s | `transform: translate(-50%, 0)` via `cubic-bezier(0.16, 1, 0.3, 1)` (silky decel) + opacity 0→1 |
| **Drain** | `var(--toast-duration)` = 6s | `.progress::after` `transform: scaleX(1) → 0` linear |
| **Hover pause** | — | `.progress::after { animation-play-state: paused }` while user hovers (read time) |
| **Auto-dismiss** | At 6s exactly | JS flips `data-open="false"` → slide out (0.35s) + drain freezes at 0% |
| **Manual dismiss** | Instant | X click → `data-open="false"` + `clearTimeout` |

### Why scaleX instead of width
- GPU-composited (no layout reflow)
- 60fps reliably even on low-end mobile
- `will-change: transform` hints the layer hoist

---

## 6. Pro card bump effect

`window.applyProBump()` is defined elsewhere in `v2.html`. It:

1. Writes `localStorage.setItem('proBumpStart', String(Date.now()))`
2. Re-renders the Pro card with bumped pricing (`$15/mo` vs `$20/mo`)
3. Renders the orange `$60 off · Ends in MM:SS` chip in the summary row + Pro card
4. Starts a 1-second `setInterval` that re-renders the countdown until expiry (60 min later)
5. After 60 minutes elapsed, auto-reverts to base pricing

The toast doesn't drive this — it's surfaced *because* the bump happened. The bump itself is driven by `?bumped=1`.

---

## 7. A11y

- `role="status"` + `aria-live="polite"` on toast — screen readers announce the message non-disruptively
- Close button has explicit `aria-label="Dismiss"` (icon-only button)
- Progress bar has `aria-hidden="true"` (purely decorative)
- `prefers-reduced-motion` consumers may want a guard:
  ```css
  @media (prefers-reduced-motion: reduce) {
    .reactivate-toast,
    .reactivate-toast .progress::after { transition: none; animation: none; }
  }
  ```

---

## 8. Production checklist

1. **Move `applyProBump` to authoritative source** — currently mutates DOM + localStorage. In production, derive from server session state (geo-aware: Tier 1 = flat $60, % markets = 50% off equivalent). Toast copy should pull from i18n with the same audience param.
2. **Sync drain duration with JS auto-dismiss** — single-source via `getComputedStyle` (see §4 example).
3. **Prevent re-firing on refresh** — once shown, store a flag (`sessionStorage.reactivateToastShown`) so a reload of `?toast=reactivated` doesn't replay the toast. Currently fires every time the URL has the param.
4. **Strip URL after consumption** — `history.replaceState(null, '', '/v2.html')` after firing so the params don't linger.
5. **Audience-aware copy** — current copy is fixed string. For % markets the message should say "**50% off** reactivated…" not `$60`. Bind to the same `audience` param used in the rest of v2.
6. **Throttle reactivations server-side** — UI says "just this once" but the page is open to anyone hitting the URL with `?bumped=1`. Server should reject the second reactivation attempt.

---

## 9. Asset map

- **Icons:** Phosphor (`ph-fill ph-check-circle`, `ph ph-x`) — loaded via existing CDN in v2.html head
- **Font:** Inter (already in v2.html)
- No external images or new dependencies

---

## 10. Versioning

| Date | Change |
|---|---|
| 2026-05-12 | Initial reactivation flow shipped. Dashboard `Reactivate and Upgrade` routes to `v2.html?bumped=1&toast=reactivated`. Toast positioned top-right, slide from top. |
| 2026-05-12 | Toast direction: slides from right edge. |
| 2026-05-12 | Toast moved to top:24px (above main nav, closer to viewport top). Close X removed for initial spec. |
| 2026-05-12 | **Toast redesigned per Figma 6603:5977** — repositioned to center-bottom, dark `#0A1925` surface, 320px width, green check-circle icon, restored dismiss X, slides from below. |
| 2026-05-12 | Single-line heading via `width: auto` + `white-space: nowrap`. Body paragraph removed — heading only. |
| 2026-05-12 | Bottom progress bar added — drains 100→0% over 6s in sync with auto-dismiss timer. |
| 2026-05-12 | **Handoff-ready refactor** — extracted `--toast-duration`, `--toast-bg`, `--toast-track-bg`, `--toast-fill-bg` custom properties. Added `:hover { animation-play-state: paused }` on drain. Progress fill switched from orange gradient to monochrome `rgba(255,255,255,0.55)` to match dark surface. |
