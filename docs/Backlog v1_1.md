## Backlog v1.1 (iOS-first, “real blocker” from day 1)

**Status:** Implementation-ready story list aligned to **Rules Engine Spec v1.1** and **Data Model Spec v1.1**.
**Primary goal:** Ship an iOS app that **actually blocks apps** via Screen Time APIs (FamilyControls / DeviceActivity / ManagedSettings) and uses your **Focus Bank + Unlock Gate + Quests + Emergency** loop.

**v1.2 monetization override (applies to implementation):**
* Tiers are **Free + Pro** only.
* Credit unlock durations remain **5/15/30** in engine; UI/plan gating shows **Free = [5]**, **Pro = [5,15,30]**.
* Free cap for distracting apps = **3**.
* On downgrade (trial end/lapse): keep highest-priority mode active, lock other modes, enforce 3-app cap, hide 15/30, honor active grants until expiry.
* Pro purchase options: monthly + annual (trial) + lifetime (non-consumable).
* Trial/paywall triggers: 4th distracting app, 2nd mode, 15/30 unlocks, Strict/Hard enable.

> **Out of scope (v1.1):** Partner approvals + Tokens (server-authoritative token ledger) + Cloud sync.
> These return in later versions once iOS enforcement + retention are stable.

---

# Global constants (v1.1 “truth”)

These appear throughout acceptance criteria:

* **Day boundary:** 04:00 local time
* **Credits rollover cap:** 10 credits
* **Streak qualifies:** either

    1. focus session **≥ 20 min completed**, OR
    2. **2 valid habit completions** (awards not suspended)
* **Streak bonus (once/day at first qualification):** +0/+5/+10/+15/+20 (cap at day 5+)
* **Quest unlock grant:** 5 minutes (fixed v1.1)
* **Quest per-app daily limit:** 2 quest grants / app / day
* **Quest cooldown per app:** 15 minutes between quest grant starts
* **Emergency unlock:** 1/day global, 60s delay, 5 min duration, cannot use same app twice consecutively
* **Focus credits formula:** `floor(duration/25)*10 + (duration>=50 ? +5 : 0)` if completed and ≥20
* **Habit reward default:** 5 credits; 1 completion/habit/day
* **Anti-gaming:** >20 habit completions in 60 seconds → suspend habit awards until next day
* **AppId format:**

    * iOS: `iosToken:<opaqueEncodedToken>`
    * Android (later): `android:<packageName>`

---

# Milestones (v1.1)

## M0 — iOS “Real Blocker” Core Loop (ship this)

* Local DB + rules engine
* Onboarding, Modes, Focus, Habits
* Unlock Gate + Quests + Emergency
* **iOS Screen Time enforcement** (authorization, app selection, scheduled shielding, custom shield UI)
* Weekly review + minimal settings

## M1 — Polish + Monetization (optional but recommended for launch)

* Paywall + entitlements gating (local-only)
* Strong onboarding clarity, tooltips, better UX
* Diagnostics export, retention cleanup, crash reporting
* Compliance screens (privacy/terms) + store-ready permission disclosures

## Post-v1.1 — Android port

* Accessibility + overlay enforcement
* Android installed apps picker
* Keep shared core engine & UI

---

# Epic E0 — Project foundation & dev workflow (M0)

### **E0-01 — Repo scaffold + TypeScript conventions (P0)**

**Acceptance**

* Standardized folders: `src/domain`, `src/data`, `src/ui`, `src/native`, `src/telemetry`, `src/tests`
* Lint + typecheck + unit tests in CI
* `SPEC/` folder added containing:

    * `rules_engine_v1_1.md` (reference)
    * `data_model_v1_1.md` (reference)
    * `backlog_v1_1.md` (this file)

---

### **E0-02 — Expo dev build baseline (P0)**

**Acceptance**

* App runs on iOS simulator + physical device via dev builds
* Feature flags exist and default:

    * `ios_screentime_enabled = true` (for v1.1)
    * `android_enforcement_enabled = false`
    * `billing_enabled = optional` (default false until M1)

---

### **E0-03 — App init sequence + Safe Mode (P0)**

**Acceptance**

* Startup order:

    1. open DB (in App Group container on iOS)
    2. ensure singleton rows
    3. `ensureDayState(now)`
    4. mount navigation
* If DB fails → Safe Mode screen with export diagnostics button (stub ok in M0)

---

### **E0-04 — Time/day_id utility + tests (P0)**

**Acceptance**

* `computeDayId(tsUtcMs, timezoneId)` matches spec exactly
* Tests cover 03:59 vs 04:00, DST sanity

---

# Epic E1 — Local data layer (SQLite) & repositories (M0)

> Must match **Data Model Spec v1.1** exactly.

### **E1-01 — Implement SQLite schema + migration v1 (P0)**

**Acceptance**

* Creates all v1.1 tables (including `entitlement_state`, `app_info`, `distracting_app`)
* Sets `meta.schema_version = "1"`
* On iOS, DB stored in App Group shared container

---

### **E1-02 — Singleton repos (P0)**

Repos:

* `EconomyRepo`, `SettingsRepo`, `EntitlementRepo`, `EmergencyRepo`, `FocusStateRepo`
  **Acceptance**
* Idempotent `ensureInitialized()`
* All writes accept `tx`

---

### **E1-03 — App catalog repos: app_info + distracting_app (P0)**

**Acceptance**

* `AppInfoRepo.upsert(appId, displayName?, platform)`
* `DistractingAppRepo.add/remove/list`
* Can store “unknown” display name and allow user alias later

---

### **E1-04 — Mode repos (CRUD + tombstones) (P0)**

**Acceptance**

* Mode + windows + blocked apps tables implemented
* Delete uses tombstone (`deleted_ts_utc_ms`)

---

### **E1-05 — Focus + habit repos (P0)**

**Acceptance**

* Focus session lifecycle stored
* Habit completion unique per habit/day enforced

---

### **E1-06 — Unlock attempt + grant repos (P0)**

**Acceptance**

* Attempt creation with snapshot JSON
* `createOrExtendGrant` enforces `unlock_grant_current` invariant
* Query active grant by app_id at time now

---

### **E1-07 — Quest repos + daily limits (P0)**

**Acceptance**

* Quest sessions recorded; daily limits updated ONLY on successful grant

---

### **E1-08 — DaySummary repo (P0)**

**Acceptance**

* Atomic increment helpers (credits earned/spent, unlock counts, focus minutes)
* Never deletes day_summary rows

---

### **E1-09 — Event log (append-only) (P0)**

**Acceptance**

* `EventLogRepo.append(tx, type, payload, nowMeta)`
* Strictly append-only except retention cleanup

---

# Epic E2 — Rules engine core (M0)

> Pure domain logic backed by repos + transactions.

### **E2-01 — ensureDayState + rollover (P0)**

**Acceptance**

* On day change:

    * carry min(10, balance)
    * reset daily flags
    * emit `DAY_ROLLOVER`

---

### **E2-02 — Streak qualification + bonus awarding (P0)**

**Acceptance**

* Qualify via focus ≥20 completed OR 2 valid habits
* Award streak bonus once/day after first qualification
* Writes day_summary.qualified + first_qualified_ts
* Emits `STREAK_QUALIFIED` and `STREAK_BONUS_AWARDED`

---

### **E2-03 — Focus credits earning (P0)**

**Acceptance**

* Uses formula exactly
* End early awards 0 and does not qualify

---

### **E2-04 — Habit credits earning + anti-gaming suspension (P0)**

**Acceptance**

* > 20 completions in 60s triggers suspension until next day
* Suspended completions award 0 and do not qualify day

---

### **E2-05 — Mode activation + precedence evaluation (P0)**

**Acceptance**

* Cross-midnight windows supported
* Manual overrides respected
* Precedence:

    1. highest priority
    2. tie → higher strictness
    3. tie → latest updated_ts

---

### **E2-06 — evaluateAppAccess(appId, now) (P0)**

**Acceptance**
Order:

1. ensureDayState
2. focus session strict blocking
3. active grant allows
4. effective mode block decision
   Returns:

* ALLOW/BLOCK + reason
* options snapshot (when blocked)

---

### **E2-07 — UnlockAttempt lifecycle API (P0)**

**Acceptance**

* When gate shown → create attempt + snapshot + event
* On select option → update attempt + event
* On grant/deny/cancel → outcome updated + events

---

### **E2-08 — Credits unlock grant (P0)**

**Acceptance**

* 5/15/30 min options at costs 10/25/45 (or tier-configurable later)
* Atomic:

    * spend credits
    * create/extend grant
    * mark attempt GRANTED
    * update day_summary
    * events emitted

---

### **E2-09 — Quest unlock rules (P0)**

**Acceptance**

* Quest grant always 5 minutes
* Daily limit + cooldown enforced
* Failure/cancel does not consume limit

---

### **E2-10 — Emergency unlock rules (P0)**

**Acceptance**

* 1/day, 60s delay, 5 min duration
* Cannot use same app twice consecutively
* Creates delayed grant

---

### **E2-11 — Truth-table test suite (P0)**

**Acceptance**
Automated tests for:

* day boundary
* streak/bonus once/day
* quest limits/cooldowns
* grant extension
* emergency rules
* focus session blocks unlocks

---

# Epic E3 — UX shell + onboarding (M0)

### **E3-01 — Navigation shell (P0)**

Tabs:

* Modes
* Focus
* Unlocks
* Progress
* Settings

---

### **E3-02 — iOS-first onboarding flow (P0)**

**Acceptance**
Onboarding completes in < 2 minutes and ends with:

* Screen Time authorization requested
* User selects distracting apps (picker)
* A default Work mode is created + scheduled
* User sees “You’re protected now” confirmation

---

### **E3-03 — Permissions explanation screens (P0)**

**Acceptance**
Screens explain:

* Why Screen Time access is required
* What data is used (app tokens only, local-first)
* How to disable/reenable

---

### **E3-04 — “Distracting Apps” setup in onboarding (P0)**

**Acceptance**

* Saves selected iOS tokens into:

    * `app_info` (best-effort display name)
    * `distracting_app`
* Explains these are used for:

    * default focus blocklist
    * templates

---

### **E3-05 — Graceful degraded mode if permission denied/revoked (P0)**

**Acceptance**

* If Screen Time permission denied:

    * App remains usable (Focus Bank, habits, focus timer, demo gate)
    * Shows banner “Blocking is OFF” and CTA to enable

---

# Epic E4 — Modes + “distracting apps” UX (M0)

### **E4-01 — Distracting Apps management screen (P0)**

**Acceptance**

* User can:

    * open native iOS picker to add/remove
    * see count selected
    * optionally rename (user alias)
* Explains: “These are your default blocked apps.”

---

### **E4-02 — Mode list + create/edit/delete (P0)**

**Acceptance**

* CRUD with tombstones
* Free tier limitation (if billing later) can be stubbed with local tier

---

### **E4-03 — Mode schedule editor (cross-midnight) (P0)**

**Acceptance**

* Weekdays mask + start/end HH:MM
* End < start treated as crossing midnight
* Validation for zero-length

---

### **E4-04 — Mode blocked apps editor (iOS picker) (P0)**

**Acceptance**

* For each mode, user selects apps to block using iOS picker
* Stores as `mode_blocked_app(app_id)`
* If displaying names is limited, show “X apps selected” + aliases where provided

---

### **E4-05 — Mode precedence UI explanation (P1)**

**Acceptance**

* When overlaps occur, UI explains which mode is effective and why (priority/strictness)

---

### **E4-06 — Manual override controls (P1)**

**Acceptance**

* AUTO / FORCED_ON / FORCED_OFF
* Forced on can have duration
* Changes persist and affect enforcement (see iOS epic)

---

# Epic E5 — Focus sessions (M0)

### **E5-01 — Focus timer start/end (P0)**

**Acceptance**

* Start session with 25/50/75
* End early = no credits

---

### **E5-02 — Focus blocked apps selection (P0)**

**Acceptance**

* Default focus blocklist = `distracting_app` list at session start
* Stored in `focus_session_blocked_app`

---

### **E5-03 — Focus completion awarding (P0)**

**Acceptance**

* Calls engine to award credits + update streak

---

### **E5-04 — Focus history (P1)**

**Acceptance**

* Shows sessions by day, credits earned, completed/ended early

---

# Epic E6 — Habits (M0)

### **E6-01 — Habit CRUD (P0)**

**Acceptance**

* Create/edit/activate/deactivate/tombstone delete

---

### **E6-02 — Daily habit completion (P0)**

**Acceptance**

* One completion per habit/day; handles uniqueness error gracefully

---

### **E6-03 — Habit rewards + suspension UX (P0)**

**Acceptance**

* Displays “Habit awards paused until tomorrow” when suspended

---

### **E6-04 — Habit streak display (P1)**

**Acceptance**

* Per-habit streak shown (visual only)

---

# Epic E7 — Unlock Gate + Quests + Emergency UX (M0)

### **E7-01 — Unlock Gate screen (P0)**

**Acceptance**

* Takes `appId`
* Creates unlock_attempt
* Shows:

    * app label (if known) or appId
    * current credits
    * mode strictness (if known)
    * options with disabled reasons

---

### **E7-02 — Credits unlock flow (P0)**

**Acceptance**

* Choose 5/15/30
* Confirm spend
* On success: grant created and UI shows unlock countdown

---

### **E7-03 — Breathing quest (P0)**

**Acceptance**

* 45 seconds uninterrupted
* On completion: quest grant 5 minutes (respect limits/cooldown)

---

### **E7-04 — Copy-text quest (P0)**

**Acceptance**

* Phrase 8–14 words
* 90 seconds, 2 tries
* Failure does not consume daily quest limit

---

### **E7-05 — QR key setup + QR quest (P1)**

**Acceptance**

* Key setup stored in settings_json
* Strict mode key-change cooldown 24h
* QR quest unlock grants 5 minutes

---

### **E7-06 — Emergency unlock UX (P0)**

**Acceptance**

* Confirm dialog + 60s countdown
* Shows daily limit status and “not same app twice” rule

---

### **E7-07 — Unlocks tab: active grants list + countdown (P0)**

**Acceptance**

* Lists active grants with time remaining
* Extending grant updates same entry

---

### **E7-08 — Unlock attempt history (P1)**

**Acceptance**

* Group by day_id
* Show outcome and denial reasons

---

# Epic E8 — iOS Screen Time enforcement (THE “real blocker” epic) (M0)

> This is what makes the app a **real blocker**.

### **E8-01 — Entitlements + capabilities checklist (P0)**

**Acceptance**

* Project includes required capabilities and entitlements for:

    * FamilyControls authorization
    * DeviceActivity monitoring
    * ManagedSettings shielding
    * App Group shared container
* Repo contains a `docs/ios_entitlements_checklist.md`

---

### **E8-02 — iOS authorization flow (FamilyControls) (P0)**

**Acceptance**

* UI requests authorization
* Handles: granted / denied / restricted
* Stores blocking-enabled state in local settings

---

### **E8-03 — iOS app picker integration (FamilyActivityPicker) (P0)**

**Acceptance**

* Native picker returns selected application tokens
* JS receives tokens as encoded string AppIds:

    * `iosToken:<opaqueEncodedToken>`
* Persist to DB:

    * `app_info` upsert
    * `distracting_app` update
    * `mode_blocked_app` update when used for a mode

---

### **E8-04 — Token encoding/decoding strategy + tests (P0)**

**Acceptance**

* Deterministic encode/decode in Swift
* Extension can decode AppId strings to `ApplicationToken`
* Unit test: encode→decode roundtrip for sample token objects (as feasible)

---

### **E8-05 — DeviceActivity monitor manager (app side) (P0)**

**Acceptance**

* When modes change, the app:

    * registers/updates monitoring schedules for each mode window
    * can stop monitoring when mode deleted/tombstoned
* Handles cross-midnight and day-of-week windows correctly (implementation-specific, but behavior must match mode schedule)

---

### **E8-06 — DeviceActivityMonitor extension applies shields on interval start/end (P0)**

**Acceptance**

* On interval start:

    * reads modes + blocked apps from SQLite (App Group DB)
    * computes active blocked set (union of mode_blocked_app for active modes)
    * applies ManagedSettings shield for that set
* On interval end:

    * clears shield (or recomputes remaining active modes if overlap)

---

### **E8-07 — Custom shield UI (ManagedSettingsUI) (P0)**

**Acceptance**

* Shield screen clearly explains:

    * “This app is blocked”
    * “Open Earned to unlock”
* Provides buttons:

    * Close app
    * “Unlock options” (button triggers delegate; see next story)

---

### **E8-08 — ShieldActionDelegate writes “last blocked app” keys (P0)**

**Acceptance**

* When shield shown or unlock button pressed:

    * writes App Group KV:

        * `ios_last_blocked_app_id`
        * `ios_last_blocked_ts_utc_ms`
        * optional mode_id
* Returns system action:

    * `.close` (recommended) so user returns to Home and can open Earned

---

### **E8-09 — App deep-links user into Unlock Gate using App Group KV (P0)**

**Acceptance**

* On app launch/resume:

    * if last blocked app key exists and is recent (≤10 minutes):

        * navigate directly to Unlock Gate for that appId
        * clear keys after navigation

---

### **E8-10 — Unlock grants actually unshield apps temporarily (P0)**

**Acceptance**

* When an unlock grant is created/extended:

    * iOS enforcement layer ensures the app is accessible during grant window
    * after grant ends, app becomes blocked again if mode still active

**Implementation requirement**

* Must not rely on app being foreground to re-block.
* Recommended: schedule a **DeviceActivity monitor for the grant interval** (non-repeating) or another system-backed mechanism.

---

### **E8-11 — Grant-expiration re-block correctness tests (P0)**

**Acceptance**

* Documented manual test checklist + at least one automated integration test where feasible:

    * Grant for 5 minutes → app accessible
    * After 5 minutes → app blocked again (shield shown)

---

### **E8-12 — “Blocking health” status panel (P1)**

**Acceptance**

* Settings shows:

    * Screen Time permission state
    * App Group configured
    * Monitoring running
    * Last shield event timestamp

---

# Epic E9 — Monetization & entitlements (optional for launch) (M1)

> Uses `entitlement_state` locally. No cloud.

### **E9-01 — Local tier gating system (P1)**

**Acceptance**

* `entitlement_state.tier` drives:

    * max modes (FREE: 1)
    * strictness cap (FREE: Gentle; PRO: Hard)
    * quest availability (FREE: breathing only; PRO: copy + QR)

---

### **E9-02 — Paywall UI (P1)**

**Acceptance**

* Paywall explains Pro features
* Does not mention tokens/partners

---

### **E9-03 — Billing integration (RevenueCat) (P2)**

**Acceptance**

* If enabled:

    * updates entitlement_state on purchase events
* If disabled:

    * app still functional with FREE tier

---

# Epic E10 — Progress, weekly review, retention UX (M0)

### **E10-01 — Progress dashboard from day_summary (P0)**

**Acceptance**

* Shows:

    * focus minutes
    * credits earned/spent
    * unlock counts (credits/quest/emergency)
    * streak count

---

### **E10-02 — Weekly review (local-only) (P0)**

**Acceptance**

* Aggregates last 7 day_summary rows
* Shows “hardest hour” computed from unlock_attempt timestamps

---

### **E10-03 — Local notifications (opt-in) (P1)**

**Acceptance**

* Reminder to “Earn your first credits today”
* Optional bedtime reminder if Sleep mode exists
* Off by default until opt-in

---

# Epic E11 — Settings, privacy, compliance, quality (M0/M1)

### **E11-01 — Settings screen (P0)**

**Acceptance**

* Edit:

    * quest enabled types (tier-gated)
    * (optional) credit costs (tier-gated)
* Read-only display:

    * emergency rules constants

---

### **E11-02 — Retention cleanup job (P0)**

**Acceptance**

* Deletes:

    * event_log > 30 days
    * unlock_attempt/grant/quest rows > 30 days
* Keeps day_summary always

---

### **E11-03 — Privacy policy + terms screens (P0)**

**Acceptance**

* In-app text includes:

    * local-first storage
    * Screen Time usage explanation
    * what is NOT collected/sent

---

### **E11-04 — Diagnostics export (P1)**

**Acceptance**

* User-initiated export JSON includes:

    * app version/build
    * economy_state snapshot
    * active grants
    * last 100 event_log entries

---

### **E11-05 — Crash reporting (P2)**

**Acceptance**

* Crash reports enabled (provider choice later)
* No sensitive app lists uploaded

---

# Epic E12 — Launch operations (M1)

### **E12-01 — Store-safe copy + screenshots storyboard (P1)**

**Acceptance**

* Avoid medical claims
* Clearly explains “real blocking” + “unlock by earning”

---

### **E12-02 — App Review readiness checklist (P1)**

**Acceptance**

* Documented checklist includes:

    * why Screen Time access is needed
    * user benefit explanation
    * how to disable
    * data handling and privacy disclosures

---

# Epic E13 — Android port (post-v1.1, placeholder)

Mark these **P3** (not for iOS launch):

* Android installed apps list module
* Accessibility service + overlay gate
* Reuse shared rules engine + UI
* Android permission disclosures

---

## Definition of Done (applies to every story)

A story is “Done” only if:

1. No invented requirements vs v1.1 specs
2. SQLite mutations are atomic where required
3. Required events are written to `event_log`
4. Tests exist for the story’s critical behavior
5. UX includes visible error states (no silent failures)

---
