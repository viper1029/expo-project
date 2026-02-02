# `docs/v1.1/02_IOS_ENFORCEMENT_SPEC.md`

## Goal

Ship iOS-first with **real blocking** using Screen Time APIs, while keeping Expo for the main app UI/business logic.

## Required iOS capabilities

* **App selection** (FamilyControls picker)
* **Shielding** (ManagedSettings)
* **Scheduling** (DeviceActivity)
* **Extensions**:

    * DeviceActivityMonitor extension (to toggle shields at schedule boundaries + handle grant expiry)
    * Shield UI / Shield Action extensions (optional branding + messaging)

## Entitlement requirement

Using FamilyControls requires the **Family Controls entitlement** for the app + related extensions. ([Apple Developer][4])
(Plan for entitlement approval as a launch gating item.)

---

## The “no redirect” constraint (design constraint)

There is **no supported way** for the Screen Time shield extension to open the main app. ([Apple Developer][1])
So: v1.1 shield UX must not depend on “Open Earned Access” buttons.

---

## Enforcement architecture (recommended)

### System of record

* **Rules engine + DB** remain local-first (your existing architecture).
* iOS enforcement reads:

    * “Which apps should be shielded right now?”
    * “Which apps are temporarily allowed (active grants)?”
    * “When do grants expire?”

### Shield set computation

At any time we compute:

```
shieldedApps = blockedAppsByEffectiveMode - appsWithActiveGrant
```

**Blocked apps** are from the effective mode (schedule + overrides + precedence).
**Active grants** come from `unlock_grant_current` + ends_ts > now.

### How shields are applied

* When a mode becomes active (or overrides change), update `ManagedSettingsStore.shield.applications` with `shieldedApps`.
* When a grant is created for app X:

    * remove X from shields (allow it)
    * schedule a re-shield at `grant_end_ts`

---

## Grant expiry: how to reliably re-block

### Preferred mechanism: one-off DeviceActivity monitoring

Use `DeviceActivityCenter.startMonitoring` with a schedule that ends at the grant expiration time, and in the monitor’s callback, re-apply shielding.

**Why:** your main app will usually be backgrounded while the user uses the blocked app, so you need an extension callback rather than relying on app timers.

`DeviceActivitySchedule` supports `repeats: false`, which indicates the schedule does not recur (single interval). ([Apple Developer][5])

### Fallback (best-effort)

* If a grant expiry callback is missed:

    * Ensure shields are reconciled on:

        * app foreground
        * mode boundary triggers
        * next shield event
    * This keeps you from “permanently unshielding” in edge cases.

---

## Shield UI content (v1.1)

### Required content

* App is blocked
* Which mode is blocking it (if available)
* Simple next step:

    * “Open Earned Access to unlock”
    * “Or start a Focus session to earn credits”

### Buttons

Because action space is constrained (none/close/defer), keep it simple:

* Primary: Close app
* Secondary: Defer (refresh) or none (depending on desired behavior) ([Riedel][2])

### Optional (v1.1+)

* Offer “Send me a reminder” (local notification) but only if notifications enabled.
* Do not promise auto-redirect.

---

## Data storage for iOS tokens

### Requirement

Store Screen Time selection tokens in a way that:

* survives app restarts
* is available to extensions (App Group container)

### Recommended

* Store selection data under App Group container (native)
* Mirror “selected app ids” into your SQLite for display and for Android parity

---

## Manual test checklist (iOS enforcement)

1. Select 1–2 apps in picker
2. Enable a mode schedule for “right now”
3. Open a blocked app → shield appears
4. Open Earned Access app → create 5‑minute unlock grant
5. Open blocked app → should open
6. Wait until grant expires → app should re-shield (ideally while still in foreground)
7. Confirm: after expiration, the shield shows again

---

