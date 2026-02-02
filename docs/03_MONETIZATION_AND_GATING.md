# `docs/v1.1/03_MONETIZATION_AND_GATING.md`

## Decision: 2 paid tiers + trial-first UX

You asked for 2 tiers and you’re leaning trial. Here’s the recommended structure:

* **Free baseline** (not marketed as a “tier”, just the default state)
* **Plus** (paid subscription)
* **Pro** (paid subscription)
* **Trial:** 7‑day Pro trial (optional, offered during onboarding)

This gives:

* low friction to activate real blocking (important for trust + reviews)
* a clean upgrade path
* trial conversion leverage

If you want *pure trial* (no meaningful free), you can flip one switch: after trial ends, disable enforcement + show paywall. I don’t recommend that for a blocker because it increases uninstall risk, but it’s feasible.

---

## What is gated (v1.1)

### Free baseline (must still feel like a “real blocker”)

* iOS enforcement ON (core promise)
* 1 Mode max
* Gentle strictness only
* Credits unlock 5 min only (fixed cost)
* Breathing quest only
* Emergency unlock enabled (1/day global) but with strict limits
* Basic progress (today + streak)

### Plus (core paid plan)

* Unlimited Modes
* Gentle + Strict
* Credit unlock 5/15/30
* Breathing + Copy-text quests
* Weekly review + insights (local-only)
* More settings control (e.g., choose which quests are enabled)

### Pro (premium plan)

* Everything in Plus
* Hard strictness (anti-bypass UX)
* Mode edit locks + disable cooldown protections
* Advanced scheduling features (multiple windows/day, stronger precedence tooling)
* “High-friction unlock” options (e.g., QR quest) **if/when implemented**

---

## Trial

**Offer:** 7 days of Pro
**When shown:** during onboarding after app selection (so it’s contextual: “Unlock more ways to earn access”)
**If user skips:** they remain on Free baseline.

**When trial ends:**

* Do NOT leave shields in a broken state.
* Keep Free baseline working (recommended).
* Show “Trial ended” paywall card inside Settings + on key upgrade screens.

---

## Implementation notes

* Use RevenueCat for entitlements.
* Cache tier locally with timestamp.
* Tier gates must be **local-first** (do not require network on app start).

---

## Pricing (placeholders you can revise)

I’m not locking actual price points unless you want me to, but here’s a sane starting template:

* Plus: monthly + annual
* Pro: monthly + annual
* Annual ~ 40–60% discount vs monthly

(When you decide, we’ll put product IDs and price display rules into a single constants file per your Epic E8 requirements.)

---

