# `docs/v1.1/03_MONETIZATION_AND_GATING.md`

## Decision: Free + Pro + trial-first UX

You asked for 2 tiers and you’re leaning trial. Here’s the structure:

* **Free baseline** (not marketed as a “tier”, just the default state)
* **Pro** (paid subscription)
* **Trial:** 7‑day Pro trial (optional, offered during onboarding)

This gives:

* low friction to activate real blocking (important for trust + reviews)
* a clean upgrade path
* trial conversion leverage

If you want *pure trial* (no meaningful free), you can flip one switch: after trial ends, disable enforcement + show paywall. I don’t recommend that for a blocker because it increases uninstall risk, but it’s feasible.

---

## What is gated (v1.1 / v1.2 aligned)

### Free baseline (must still feel like a “real blocker”)

* iOS enforcement ON (core promise)
* 1 Mode max
* **Up to 3 distracting apps**
* Gentle strictness only
* Credits unlock 5 min only (fixed cost)
* Breathing quest only
* Emergency unlock enabled (1/day global) but with strict limits
* Basic progress (today + streak)

### Pro (paid plan)

* Unlimited Modes
* Unlimited distracting apps
* Strict + Hard strictness (anti-bypass UX)
* Credit unlock 5/15/30
* Breathing + Copy-text quests
* Weekly review + insights (local-only)
* More settings control (e.g., choose which quests are enabled)
* Advanced scheduling features (multiple windows/day, stronger precedence tooling)
* “High-friction unlock” options (e.g., QR quest) **if/when implemented**

---

## Trial

**Offer:** 7 days of Pro
**When shown:** during onboarding after app selection (so it’s contextual: “Unlock more ways to earn access”)
**If user skips:** they remain on Free baseline.

**Recommended trial/paywall triggers:**

* Adding the **4th distracting app**
* Creating a **2nd mode**
* Choosing **15/30-minute unlocks**
* Enabling **Strict** or **Hard** mode

**When trial ends:**

* Do NOT leave shields in a broken state.
* Keep Free baseline working (recommended).
* Show “Trial ended” paywall card inside Settings + on key upgrade screens.

**Downgrade behavior (deterministic):**

* User becomes **Free immediately** (no limbo state).
* App does **not delete user data**.
* App enforces Free limits and explains what changed.

Concrete rules:

* **Modes:** keep the highest-priority mode (tie-breaker: most recently updated). Other modes stay in DB but are locked in UI and not enforced.
* **Distracting apps:** enforce cap = 3. Prompt user to pick 3 to keep, or auto-keep the 3 most recently added with a banner.
* **Unlock durations:** hide 15/30 in UI; keep only 5.
* **Active grants:** honor existing grants until they expire.

---

## Implementation notes

* Use RevenueCat for entitlements.
* Cache tier locally with timestamp.
* Tier gates must be **local-first** (do not require network on app start).
* Enforce the Free app cap in both UI and rules/validation (do not silently trim).
* On iOS, enforce the cap in the picker wrapper UI (show “Selected X/3” and disable Done if >3).

---

## Pricing (placeholders you can revise)

I’m not locking actual price points unless you want me to, but here’s a sane starting template:

* Pro: monthly + annual
* Annual ~ 40–60% discount vs monthly

**Include a lifetime option:** one-time non-consumable purchase that grants Pro.

(When you decide, we’ll put product IDs and price display rules into a single constants file per your Epic E8 requirements.)

---
