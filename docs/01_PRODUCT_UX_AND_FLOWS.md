## Product promise (store-safe)

**“Earned Access”**: instead of white-knuckling, you earn short unlock windows using Focus time + small interventions.

**Core loop (v1.1):**

* **Modes** define when/what gets blocked.
* **Focus Bank** earns credits offline.
* **Unlock Gate** spends credits or runs a quest for short access.
* **Emergency** is limited, obvious, and intentionally frictional.

## UX principles (v1.1)

1. **Clear state at a glance**

    * “What’s active?” (Mode, strictness)
    * “Why blocked?” (Reason + next step)
    * “How to unlock?” (Options, costs, disabled reasons)
2. **Fast path for daily use**

    * Start Focus in ≤ 2 taps
    * Unlock in ≤ 10 seconds (when eligible)
3. **No surprises**

    * If something is disabled (cooldown, insufficient credits), explain why and when it becomes available.
4. **Offline-first**

    * Credits, quests, emergency, and mode schedule must still behave correctly offline (except network-only unlock types like tokens/partner, if/when present).

---

## iOS-first reality check (critical for “real blocker” positioning)

When iOS shielding is active, **you cannot rely on the shield to “open your app”** via a button. Apple explicitly states there’s **no supported way** for the Screen Time shield extension to open the main app. ([Apple Developer][1])

**Implication for UX (v1.1 decision):**

* The shield can block and show your branding + instructions,
* but the user will typically **swipe Home and open your app manually** to unlock (or optionally use Shortcuts automation later).

Also, the shield action responses are essentially limited to `none`, `close`, and `defer` behaviors (refresh). ([Riedel][2])

**Optional mitigation (v1.1+):**

* You *can* use a **local notification** as a “tap to open blocker” nudge after a block event (this is a known workaround pattern), but it requires notification permission and isn’t instant/guaranteed. ([Stack Overflow][3])

---

## Detailed flows

### Flow A — First-run onboarding (iOS-first)

**Goal:** user completes setup in < 2 minutes and experiences “real blocking” immediately.

1. **Welcome + value**

    * “Choose distracting apps”
    * “Set a schedule”
    * “Unlock with Focus Bank”
2. **Trial/paywall (recommended placement)**

    * Show **“Start 7‑day Pro trial”** *with a clear “Not now”* option.
    * If user skips: continue with **Free baseline** (limited but real blocking).
3. **Select distracting apps (iOS)**

    * Use iOS Screen Time picker (FamilyControls).
    * User selects apps/categories.
    * Store selection locally (and in App Group container so extensions can read).
4. **Create first Mode from template**

    * “Work” default schedule (e.g., weekdays 9–5)
    * Strictness default **Gentle** on Free
5. **Explain unlock options**

    * Credits unlock + quests + emergency (and show disabled “future” options if any)
6. Land in Tabs (Modes / Focus / Unlocks / Progress / Settings)

**Success condition:** User can attempt to open a blocked app and see the shield that confirms blocking is active.

---

### Flow B — Daily use

1. User sees Home widgets:

    * Credit balance
    * Streak
    * Active mode
2. User starts Focus session (optional but encouraged)
3. Credits accrue; user can unlock later

---

### Flow C — “Blocked app attempt” (iOS real blocker)

1. User taps blocked app
2. iOS shows shield
3. Shield message:

    * “Blocked by Work Mode”
    * “To unlock: open Earned Access and choose an unlock option.”
4. User closes app (system “Close”)
5. User opens Earned Access app → Unlock Gate
6. Earned Access app grants a temporary allow window
7. User opens blocked app successfully during grant
8. After grant ends, app becomes blocked again automatically (enforcement spec covers timing)

---

### Flow D — Unlock Gate (in-app)

**Inputs:** appId (+ modeId optional)
**Screen shows:**

* “Why blocked” reason (Mode, Focus session active, etc.)
* Credit balance + streak
* Unlock options list:

    * Credit unlock (5/15/30)
    * Breathing quest (45s)
    * Copy-text quest (90s)
    * Emergency (if eligible)
    * Token/Partner shown only if implemented and eligible

**Behavior rules:**

* Disabled reasons always visible (e.g., cooldown, daily limit).
* On success:

    * Show “Unlocked for X minutes” + end time.

---

## App selection: “Can user define distracting apps?”

### iOS (v1.1)

Yes, via the Screen Time selection UI (FamilyControls).
**Important:** iOS does not allow reading “all installed apps” freely; the system picker is the supported path.

### Android (later)

List installed apps via native module; let user choose, then enforce via overlay.

---

## UX unknowns that v1.1 resolves

✅ **Shield → app redirect:** not possible via supported APIs; do not build UX that depends on it. ([Apple Developer][1])
✅ **Real blocker on day 1:** achieved by iOS shielding + in-app unlock creation (manual open).
✅ **User-defined distracting apps:** yes on iOS via picker, Android later via installed-app list.

---

