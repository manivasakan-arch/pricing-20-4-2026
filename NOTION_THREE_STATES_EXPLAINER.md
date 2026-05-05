# Dashboard Header — 3 States Explained

> Sub-page for the **Pricing page Update April** Notion doc.
> Live demo: https://manivasakan-arch.github.io/pricing-20-4-2026/dashboard-flash-sale-expired.html
> Engineering handoff: https://github.com/manivasakan-arch/pricing-20-4-2026/blob/manivasakan-arch/dev-api-changes/DASHBOARD_FLASH_SALE_EXPIRED_HANDOFF.md

---

## 🟧 Active

**When:** user submits feedback (or closes feedback modal) → discount unlocked. 60-min countdown begins.

**Nav UI**
```
[$60 off · ⏱ Ends in 59:51]   [🔥 Trial: 4 days left  🚀 Upgrade]
```

**Behavior**
- Pulsing orange chip with live `MM:SS` timer ticking down
- `up-chip-bob` + `up-chip-glow` ripple animations call attention
- Trial/Credits chip cycles next to Upgrade rocket CTA
- Click `Upgrade` → checkout with `?promo=PROANNUAL60`

**Lifespan:** **60 minutes.** Then auto-flips to Expired.

---

## 🟥 Expired

**When:** 60-min countdown elapsed without user upgrading. Discount window closed.

**Nav UI**
```
⚡ Missed $60 off Pro Annual →   [↻ Reactivate and Upgrade]
                                    ↳ One reactivation left
                                       Tap to bring back $60 off
                                       Last time we can offer it.
```

**Behavior**
- Orange "Missed $60 off Pro Annual" message + bouncing arrow drawing eye to CTA
- `Reactivate and Upgrade` button with shimmer sweep
- Hover → urgency tooltip: "One reactivation left · Last time we can offer it."
- Click → re-flips header to Active (timer restarts) + routes to checkout
- One reactivation per user (server-enforced); second time stays Expired permanently

**Lifespan:** until clicked (one-shot) OR session ends.

---

## 🟨 Close Nudge

**When:** user opens pricing modal → closes feedback popup **without submitting feedback.** They saw the offer but didn't engage.

**Nav UI**
```
[🎁 Share feedback and get $60 · Claim Discount]   [🔥 Trial: 4 days  🚀 Upgrade]
                                  ↑ underlined inline button
```

**Behavior**
- Soft-glow chip (`nudge-glow` 2.4s ∞) — gentler than Active, not yelling
- "Claim Discount" inline-text-button (underlined) reopens feedback modal — 2nd chance to submit + unlock $60
- Standard `🚀 Upgrade` CTA on the right (same Trial/Credits cycler as Active) for users who want to upgrade without engaging with feedback
- Click `Claim Discount` → reopens feedback testimonial picker → submit → flips header to Active
- Click `Upgrade` → checkout at full price (no discount applied)

**Lifespan:** session-long until claimed (→ Active) or upgraded.

---

## State transition diagram

```
                              ┌──────────────┐
   user opens pricing  ──→    │   feedback   │
                              │    modal     │
                              └──────┬───┬───┘
                 closes without      │   │  submits feedback
                 submitting          │   │
                              ┌──────┘   └──────┐
                              ▼                 ▼
                        ┌──────────┐      ┌──────────┐
                        │  Close   │      │  Active  │
                        │  Nudge   │      │ (60 min) │
                        └─────┬────┘      └─────┬────┘
                              │                 │
                clicks Claim  │                 │ 60 min elapsed
                Discount      │                 │ no upgrade
                              │                 ▼
                              │           ┌──────────┐
                              │           │ Expired  │
                              │           │ (1 chance│
                              │           │ to react)│
                              │           └─────┬────┘
                              │                 │ clicks Reactivate
                              ▼                 ▼
                              ┌─────────────────┐
                              │     Active      │
                              │ (timer restart) │
                              └─────────────────┘
```

---

## Summary table

| State | Trigger | Discount status | CTA | Lifespan |
|---|---|---|---|---|
| **Active** | User submitted/closed feedback | Live 60-min countdown | `🚀 Upgrade` | 60 min, then → Expired |
| **Expired** | 60-min timer elapsed | Closed, one revival left | `↻ Reactivate and Upgrade` | Until clicked (one-shot) |
| **Close Nudge** | User saw modal, dismissed feedback | Available, un-claimed | `Claim Discount` inline link + `🚀 Upgrade` | Session-long until claimed/upgraded |
