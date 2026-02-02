## Product Blueprint v1.1

**Status:** Freeze these decisions before coding.
**Product working name:** **Earned** (placeholder)
**Platforms:** iOS first (real OS-level blocking), Android second
**Build:** Expo + dev builds (custom dev client) + native modules/extensions as required

---

# 0) Executive summary

**Category:** Focus / Screen time / App blocking
**Core promise:** *Earn leisure. Spend credits, not willpower.*
**Core mechanic:** **Focus Bank** (credits) earned via deep work + habits → spent to unlock blocked apps for limited windows.

**Why it’s different:** It’s not “block forever” — it’s an **economy** that turns distraction into a **budget**.

**Real blocker positioning (Day 1):**

* iOS ships with **Screen Time shielding** (ManagedSettings / DeviceActivity), not “demo mode only.”
* Android enforcement follows (Accessibility + overlay gate).

**Critical iOS constraint:** the iOS shield cannot directly open/redirect into your app; unlock happens *in your app*, then user returns to the blocked app.
We design the UX around this (see §6.3).

---

# 1) Goals and non-goals

## Goals (v1 launch)

1. **Real blocking on iOS**: scheduled modes block selected apps at OS-level.
2. **Earned access loop**: focus sessions/habits earn credits → credits/quests grant unlock windows.
3. **Clarity and trust**: user always understands:

    * why an app is blocked
    * what it costs to unlock
    * when it will re-lock
4. **Local-first reliability**: rules engine works offline (except features explicitly requiring network).

## Non-goals (v1 launch)

* “Medical treatment” app positioning (no addiction/ADHD treatment claims).
* Perfect anti-cheat. The goal is behavior change via friction + structure, not a DRM system.
* Cross-device (desktop) in v1.

---

# 2) Target audience

## Primary (launch)

**Students + knowledge workers** who want:

* deep work blocks
* bedtime limits
* measurable progress
* a tool they’ll actually pay for

## Secondary (post-launch positioning)

* digital minimalists
* creators building consistency
* neurodivergent audiences (position carefully; avoid medical claims)

---

# 3) Positioning & brand

## Positioning statement

> The blocker that works like a **budget**: earn credits by doing what matters, then spend them intentionally.

## Tone

Supportive coach (not punitive cop). “You’re in control” vibe.

## Store-safe language (use consistently)

* “Reduce screen time”
* “Build healthier phone habits”
* “App blocking with schedules”
* “Focus sessions and intentional breaks”

Avoid:

* “Treat addiction”
* “Cure ADHD”
* “Unbreakable”

---

# 4) Core product loop

1. User sets **Distracting Apps** and creates **Modes** (Work / Study / Sleep).
2. When a mode is active, selected apps are **blocked** (OS-level on iOS).
3. User earns **credits** via:

    * Focus sessions (timer-driven deep work)
    * Habits (daily checklists)
4. When user wants a blocked app:

    * They open **Earned** → **Unlock Gate**
    * They unlock with: **Credits** or **Quest** or **Emergency** (partner/tokens are optional roadmap)
5. The app gives **weekly review** + streak feedback.

---

# 5) Frozen v1 behavior defaults

These are “behavior truth.” (They will also appear verbatim in Rules Engine Spec.)

## Time / day boundary

* **Day starts at 04:00 local time**
* Daily limits, streaks, and credit rollover use this boundary.

## Credits reset and streak bonus

* **Daily reset with rollover cap:** keep up to **10 credits**
* **Streak qualifies** if either:

    * ≥ 1 focus session **20+ min completed**, OR
    * ≥ **2 valid habit completions**
* **Streak bonus** awarded once/day **after the first qualifying action**:

    * day 1: +0
    * day 2: +5
    * day 3: +10
    * day 4: +15
    * day 5+: +20 (cap)

## Credit earning (focus)

* Focus earns credits only if:

    * completed, and duration ≥ 20 minutes
* Formula:

    * `floor(duration/25)*10 + (duration>=50 ? +5 : 0)`

## Quests

* Quests grant **direct unlock windows** (not credits).
* Quest unlock window: **5 minutes**
* Limits:

    * **2 quest unlocks per app per day**
    * **15-minute cooldown** per app between successful quest grants

## Emergency access

* **1/day** global
* **60-second delay** then **5-minute unlock**
* Cannot use for the **same app twice in a row**
* Always logged prominently

---

# 6) UX / UI structure and flows

## 6.1 Navigation (frozen)

Tabs:

* **Modes**
* **Focus**
* **Unlocks**
* **Progress**
* **Settings**

Global route:

* **Unlock Gate** (opens with `app_id`, optional `mode_id`)

### Where key features live (to avoid confusion)

* **Distracting Apps**:

    * Primary entry: **Modes** tab (top button: “Distracting Apps”)
    * Secondary shortcut: Settings
* **Habits**: inside **Progress** (toggle: “Overview / Habits”)
* **Unlock history + active unlocks**: Unlocks tab

## 6.2 Onboarding flow (≤ 2 minutes)

1. Goal selection (Focus / Study / Sleep / Social control)
2. Select **Distracting Apps** (iOS system picker)
3. Create first mode from template:

    * Work Mode (weekday schedule default)
    * Sleep Mode (night schedule default)
4. Explain “Earn credits → unlock intentionally”
5. Ask for permissions (iOS Screen Time authorization)

**Onboarding success criteria:**

* At least one Mode exists and is scheduled.
* At least one distracting app selected.
* User understands: credits unlock apps.

## 6.3 Blocking and unlocking: iOS “real blocker” UX

**When a blocked app is opened on iOS:**

* iOS shows the **system shield** (ManagedSettings UI).
* The shield **cannot redirect into your app**.
* The shield message instructs: “Open Earned to unlock.”

**How we make this feel smooth anyway**

* The extension writes `lastBlockedAppId` (token-based) to App Group shared storage.
* When Earned is opened, it auto-routes to:

    * `UnlockGate(lastBlockedAppId)`
      so the user doesn’t have to hunt for the right app.

**Unlock options shown in-app**

* Spend credits (5/15/30 min)
* Do a quest (5 min)
* Emergency (delayed 5 min)

After unlocking:

* The app updates the iOS shield policy so the target app becomes accessible for the unlock window.
* User returns to the blocked app and it opens normally.

## 6.4 Unlock Gate (signature screen)

When a user is blocked, the app shows one screen:

* “You’re in Work Mode”
* “Instagram costs 10 credits for 5 min”
* Options (enabled/disabled with clear reasons):

    * Credits unlock
    * Quest unlock
    * Emergency unlock
    * (Partner/tokens shown only if implemented and available)

Required user-visible elements:

* Credit balance
* Remaining unlock time if already unlocked (and “unlock extends” note)
* Reason if disabled (insufficient credits, quest limit, cooldown, etc.)

## 6.5 Focus flow

* Start focus session (25/50/75 presets)
* Optional task label
* Focus protection: focus-blocklisted apps are blocked during session and cannot be unlocked mid-session.
* Completing session awards credits; ending early awards none.

## 6.6 Progress flow

* Daily dashboard: credits earned/spent, focus minutes, unlock counts
* Weekly review: last 7 days rollup
* “Hardest hour” insight from unlock attempts

---

# 7) “Distracting apps” model

This is a **global list** called **Distracting Apps**:

* Used as the default blocklist for new modes
* Used as the default focus session blocklist

Modes still maintain their own `blocked_apps` list, but the global list is the “starting point” so UX stays simple.

### iOS app identity (important)

On iOS, app selection uses Screen Time tokens. Do **not** assume stable bundle IDs.
So we store iOS app IDs as **token handles**, e.g.:

* `iosToken:<opaque>` (exact encoding defined in Rules/Data specs)

UI display:

* If we can’t resolve a friendly app name reliably, show “Blocked app” or a user-defined label.

---

# 8) Modes and strictness

## Mode definition

A Mode has:

* name, priority
* schedule windows
* blocked apps
* strictness
* which unlock methods are allowed

## Strictness tiers

* **Gentle**

    * edits allowed
    * disable immediately
    * all unlock methods allowed
* **Strict**

    * edits locked while active
    * disable has **15-minute cooldown**
* **Hard**

    * cannot disable while active
    * edits locked while active

### Important Hard-mode safety rule (v1)

Hard mode must never produce a “no options available” dead-end.

**Decision for v1 launch:**

* Hard mode is allowed, but it **must keep Emergency available** (still 1/day + delay), even if other unlock methods are disabled.
  This prevents catastrophic lockouts (especially offline) while still feeling “hard.”

---

# 9) Monetization and pricing

You can monetize day 1 without tokens/partner, using strictness + customization.

## Recommended v1 pricing (simple, conversion-friendly)

### Free

* 1 mode
* basic focus timer
* credits + basic quests
* basic progress
* Gentle only

### Plus (subscription)

* unlimited modes
* Strict mode
* full quest set (breathing + copy text + QR key)
* weekly review improvements
* advanced credit cost customization

### Pro (subscription)

* Hard mode
* deeper analytics
* advanced schedules (multiple windows/day)
* (Optional later) partner approvals, paid token overrides

**Note:** if you decide to include partner/tokens at launch, Pro would gate those. If not, they become v1.1+ roadmap features.

---

# 10) Technical strategy

## Expo strategy

* Use **Expo dev builds** (custom dev client) from day 1.
* iOS: native Screen Time extensions + bridge.
* Android: later native module + accessibility service.

## iOS enforcement approach

* Screen Time frameworks (FamilyControls / ManagedSettings / DeviceActivity).
* Requires the appropriate Apple capability/entitlement.
* iOS shield cannot open the main app; we implement lastBlockedAppId handoff.

### Shared storage on iOS

We store the SQLite DB in an **App Group shared container** so extensions and the app can read the same data.
(This reduces “two sources of truth” risk.)

## Android enforcement (post-iOS launch)

* AccessibilityService detects foreground apps
* Overlay gate shown when blocked
* Must include compliant permission education and disclosure (later doc sections cover this)

---

# 11) Privacy commitments (v1)

* Local-first: rules engine state and event log stored locally.
* If cloud features are added later (partner/tokens), sync is minimal and user-consented.
* No selling user data.
* App usage stored as: app identifiers + timestamps (for gating and analytics). No content.

---

# 12) Success metrics (v1 launch)

* Activation: % who create a mode + select distracting apps + run first focus session within 24h
* Retention: D1 / D7 retention, weekly review opens
* Outcomes: focus minutes/day, unlock attempts/day (trend down), emergency uses/day (trend down)
* Monetization: trial→paid conversion, churn

---

# 13) Roadmap (post-launch)

High-value additions:

* Android enforcement parity
* partner approvals
* token overrides
* autopilot suggestions (non-medical coaching)
* NFC tag quest

---