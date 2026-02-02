A quick accuracy check on the “holy grail redirect” write‑up you pasted, because it affects how we design iOS v1.1 UX:

* **Correct:** Apple’s Screen Time shielding flow is intentionally restrictive. You can customize a shield UI and handle button presses via extensions, and you can “refresh/redraw” the shield by returning a *defer*‑style response. Apple explicitly describes the shield response as either **close** the shielded app or **defer and redraw** the shield. ([Apple Developer][1])
* **Not reliable / likely incorrect for App Store–safe design:** “Use a shield button to open my blocker app via URL scheme / Link” is not officially supported. An Apple engineer explicitly says launching the parent app from the ShieldAction extension **is not supported**. ([Apple Developer][2])

**Implication for v1.1 (“real blocker from day 1”):**
On iOS, you should assume the shield cannot automatically “redirect” users into your app. Your iOS v1.1 UX must work with one of these patterns:

1. **Shield = block + instruction**, then user manually opens your app to unlock (most App Store‑safe).
2. **Shield = “waiting” state** (using `.defer`/redraw) while the main app (or a backend/partner) changes state; shield updates afterward.
3. Optional later: **Shortcuts automation** as an enhancement (not required for “real blocker,” and it adds setup friction).

With that clarified, here is the **Agent Prompt Pack v1.1** updated for:

* **Expo dev builds** (still the plan)
* **iOS-first launch**
* **Real blocker from day 1** using **Screen Time APIs** (FamilyControls / DeviceActivity / ManagedSettings + required extensions)
* **App selection on iOS** via the system picker (FamilyControls)
* Keeping feature flags so you can instantly disable enforcement in emergencies

---

# Agent Prompt Pack v1.1

## Master Instructions (give to every agent)

**Project goal:** Build an iOS-first “Earned Access” blocker app using Expo dev builds with **real iOS app blocking from day 1** via Apple’s Screen Time APIs.

### Non-negotiables

1. **No invented requirements.** Implement only what the v1.1 specs/backlog state.
2. **Real iOS blocking is required in v1.1** (Screen Time shielding; not “demo mode” only).
3. **Local-first correctness.** Core loop works offline. (Network-only features must be explicitly disabled offline.)
4. **Atomic state changes (SQLite transactions)** for any correctness-critical mutation.
5. **Event log must be written** for all rules-engine decisions/mutations (auditable behavior).
6. **Time & day boundary:** day starts at **04:00 local time**. `day_id` computed exactly per spec.
7. **Feature flags** must gate enforcement + cloud behaviors so we can hot-disable if needed:

    * `ios_enforcement_enabled` (default ON in release build, but still configurable)
    * `cloud_sync_enabled` (default OFF for v1.1 unless explicitly needed)
    * `android_enforcement_enabled` (OFF; Android is after iOS)
8. **Testing:** every epic includes tests for its truth-table scenarios; CI runs typecheck + tests.

### iOS Screen Time reality (design constraint)

* The ManagedSettingsUI shield experience **cannot reliably deep-link users into the main app** from the shield. Design UX accordingly. (Unlocking must be possible by manually opening the app, and/or by “waiting” state + redraw.)

### Repository structure (must adhere)

* `src/domain/` → pure business logic + types + rules engine (JS/TS)
* `src/data/` → SQLite schema/migrations + repos + transactions
* `src/ui/` → screens/components/navigation
* `src/native/` → **native modules** + iOS Screen Time integration wrappers
* `src/ios/` → iOS-specific glue code types/helpers used by extensions (if you keep some shared)
* `src/sync/` → cloud sync (optional for v1.1)
* `src/billing/` → RevenueCat/IAP (optional for v1.1; if included, must be gated)
* `src/telemetry/` → error logging + diagnostics export
* `src/tests/` or colocated `__tests__/`

### Cross-epic interface contracts (do not break)

* `RulesEngine.evaluateAppAccess(appId, nowMeta)` must be deterministic.
* All repo mutations accept a `tx` transaction handle.
* `EventLogRepo.append(tx, type, payload, nowMeta)` is the only writer for `event_log`.
* Grants must preserve **single current grant per app** invariant.

### iOS-first requirement: App Group storage

For iOS enforcement extensions to read state, the **SQLite DB must live in an App Group container** (or state must be mirrored into App Group UserDefaults). If your v1.1 spec says “extensions read SQLite,” then DB path must be App Group path. If the v1.1 spec says “mirror minimal state for extensions,” then implement that mirror. Do not improvise.

---

# Prompt Pack v1.1

## Epic E0 — Foundation & dev workflow (Expo dev builds + iOS targets readiness)

```text
You are an autonomous coding agent implementing Epic E0 (foundation) for v1.1.

READ FIRST:
- Rules Engine Spec v1.1: time/day_id, event envelope, feature flags
- Data Model Spec v1.1: DB location requirements (App Group), PRAGMAs, singletons
- Backlog v1.1: E0 stories

SCOPE:
- Repo scaffold + TS conventions + lint/typecheck CI
- Expo dev build baseline (custom dev client)
- Feature flags module (ios_enforcement_enabled, cloud_sync_enabled, android_enforcement_enabled)
- Navigation shell + placeholder screens
- App init sequencing (DB init in App Group path, ensure singletons, ensureDayState)
- computeDayId() utility + tests
- Global error boundary + Safe Mode screen + basic diagnostics view (dev only)

REQUIRED IMPLEMENTATION DETAILS:
1) Feature flags:
   - Persist locally (DB meta table preferred).
   - Defaults:
     - ios_enforcement_enabled: true (but allow override in dev)
     - cloud_sync_enabled: false
     - android_enforcement_enabled: false

2) computeDayId(tsUtcMs, timezoneId):
   - 04:00 local boundary (if local time < 04:00 => previous date)
   - Unit tests: 03:59 and 04:00 local; DST sanity.

3) App init:
   - Determine timezoneId (device) early.
   - Initialize SQLite at correct location:
     - If iOS, use App Group container path (not Documents).
     - Android can still use normal app storage.
   - Ensure singletons exist.
   - Call ensureDayState(now).

4) Navigation:
   - Tabs: Modes, Focus, Unlocks, Progress, Settings
   - Global route: UnlockGate(appId, modeId?)
   - Add iOS-only “Screen Time Setup” screen route.

DO NOT:
- Implement business rules yet.
- Implement iOS enforcement logic yet (that is E11).

DELIVERABLES:
- Bootable app with dev build workflow
- Flags + navigation + init ordering
- computeDayId + tests
```

---

## Epic E1 — Local data layer (SQLite) & repositories v1.1

```text
You are an autonomous coding agent implementing Epic E1 (SQLite + repos) for v1.1.

READ FIRST:
- Data Model Spec v1.1: all Part A (DDL), invariants, transaction boundaries, App Group DB path rules
- Rules Engine Spec v1.1: required event types
- Backlog v1.1: E1 stories

SCOPE:
- SQLite schema + migrations exactly per Data Model Spec v1.1
- Repos: singletons, modes, windows, blocked apps, habits, completions, focus sessions, focus_state, unlock attempts, grants, daily app limits, day summary, event_log
- Transaction helper withTransaction()

CRITICAL iOS REQUIREMENT:
- DB MUST be created/opened at the App Group container path so iOS extensions can read.
- Provide a single “getDatabasePath()” utility that returns:
  - iOS: app group container db path
  - Android: normal app db path

DO NOT:
- Implement rules engine logic (E2).
- Implement iOS Screen Time APIs (E11).

TESTS:
- Verify schema created
- Verify singleton rows exist
- Verify createOrExtendGrant maintains single current grant per app and extends end time correctly
- Verify unique habit completion per habit/day
```

---

## Epic E2 — Rules engine core v1.1 (JS/TS)

```text
You are an autonomous coding agent implementing Epic E2 (Rules Engine) for v1.1.

READ FIRST:
- Rules Engine Spec v1.1 (all logic)
- Data Model Spec v1.1 (settings JSON schema, option snapshots, event payload schemas, atomic ops)
- Backlog v1.1: E2 stories

SCOPE:
- Day rollover (04:00 local), credit rollover cap, streak qualification + bonus
- Mode activation + precedence
- evaluateAppAccess() deterministic
- UnlockAttempt lifecycle + option snapshot
- Credits unlock, quests (breathing/copy/QR), emergency unlock
- Strictness edit locks + disable cooldown rules

IMPORTANT:
- iOS enforcement extensions cannot run JS. The JS rules engine is the product truth,
  but E11 must ensure iOS enforcement behavior matches the same persisted state.
  Do not add iOS-only hacks here; keep engine platform-agnostic.

DO NOT:
- Implement partner or token spend unless v1.1 spec explicitly includes them.
- Implement any native enforcement interception.

TESTS:
- Truth table tests from spec:
  - rollover at 03:59 vs 04:00 local
  - streak bonus once/day
  - quest limit + cooldown
  - emergency daily limit + same-app rule
  - grant extension invariants
  - focus session blocks unlock methods
```

---

## Epic E3 — Core UI flows v1.1 (iOS-first, Screen Time setup required)

```text
You are an autonomous coding agent implementing Epic E3 (UI core flows) for v1.1.

READ FIRST:
- Backlog v1.1: onboarding, permissions/setup screens, Unlock Gate route
- Rules Engine Spec v1.1: unlock attempt + access decisions
- iOS enforcement constraint: shield cannot deep-link into app reliably

SCOPE:
- Onboarding updated for iOS-first:
  - Explain “real blocking requires Screen Time permission”
  - Guide to Screen Time authorization (calls native module from E11)
  - Create first Mode + pick distracting apps (iOS picker integration from E11)
- Unlock Gate screen (in-app), driven by RulesEngine
- Home widgets: credits/streak/active mode
- UX copy that matches iOS reality:
  - When a blocked app is shielded, user may need to open your app manually to unlock.

DO NOT:
- Fake “auto redirect” from shield.
- Implement Android picker/enforcement.

DELIVERABLES:
- Onboarding that results in:
  - Screen Time authorization requested (if ios_enforcement_enabled)
  - At least 1 Mode created
  - Distracting apps selected (iOS picker)
- Unlock Gate fully functional in-app
```

---

## Epic E4 — Modes & app selection (iOS tokens) v1.1

```text
You are an autonomous coding agent implementing Epic E4 (Modes + iOS app selection UX) for v1.1.

READ FIRST:
- Backlog v1.1: modes CRUD, schedule editor, strictness, precedence, manual overrides, blocked apps selection on iOS
- Rules Engine Spec v1.1: mode activation and precedence
- Data Model Spec v1.1: mode tables, plus iOS selection token storage approach

SCOPE:
- Mode list + create/edit/delete (tombstones)
- Schedule windows editor (cross-midnight)
- Priority + precedence explanation UI
- Manual overrides (AUTO/FORCED_ON/FORCED_OFF) with strictness cooldown rules
- Strictness editor tier gates (if tiers exist in v1.1)
- iOS app selection:
  - Open FamilyControls picker (via E11 native module)
  - Persist selected apps/categories in the correct data model format (token-based, not assumed bundle IDs unless spec says so)

IMPORTANT:
- Avoid designing UI that assumes you have “installed apps list” on iOS from JS.
- Your UI must represent selected apps even if you only have opaque tokens. Provide “friendly names” only if the native layer provides them; otherwise show a generic label.

DO NOT:
- Implement Android app picker here.
```

---

## Epic E5 — Focus sessions v1.1

```text
You are an autonomous coding agent implementing Epic E5 (Focus sessions) for v1.1.

READ FIRST:
- Backlog v1.1: focus stories
- Rules Engine Spec v1.1: focus blocking behavior and credit earnings
- Data Model Spec v1.1: focus tables

SCOPE:
- Focus timer start/pause/end
- Focus sessions persist and award credits via RulesEngine on completion
- Focus “strict protection”: blocked apps cannot be unlocked during focus (engine behavior)
- History list

NOTE (iOS reality):
- Focus blocking in iOS enforcement is separate (E11). Here you implement the product loop and engine state.
```

---

## Epic E6 — Habits v1.1

```text
You are an autonomous coding agent implementing Epic E6 (Habits) for v1.1.

READ FIRST:
- Backlog v1.1 habits
- Rules Engine Spec v1.1: habit award + anti-gaming + qualification
- Data Model Spec v1.1: habit tables and unique constraints

SCOPE:
- Habit CRUD + daily completion
- Completion calls RulesEngine.onHabitCompleted
- Anti-gaming suspension UX + banner
- Per-habit streak display (UI only)
```

---

## Epic E7 — Unlock Gate + Quests UX v1.1 (in-app)

```text
You are an autonomous coding agent implementing Epic E7 (Unlock Gate + quests UI) for v1.1.

READ FIRST:
- Backlog v1.1: unlock gate, breathing/copy/QR quests, cooldown UI, unlock countdown, attempt history
- Rules Engine Spec v1.1: quest rules (5 min grant, 2/day/app, 15 min cooldown), no credit awards from quests
- Data Model Spec v1.1: snapshot schema and quest tables

SCOPE:
- Unlock Gate UI renders engine snapshot with disabled reasons
- Credits unlock durations (5/15/30) + confirm
- Breathing quest (45s continuous)
- Copy-text quest (phrase, 90s, 2 tries)
- QR quest setup + QR quest (if v1.1 includes it)
- Active grants list + countdown
- Unlock attempt history

IMPORTANT iOS UX NOTE:
- Users might reach Unlock Gate by opening your app manually after being shielded.
  Ensure the app surfaces “Recently blocked app” and a 1-tap path to unlock that app (uses last unlock_attempt).
```

---

## Epic E8 — Billing / tiers (only if included in v1.1)

```text
You are an autonomous coding agent implementing Epic E8 (billing) for v1.1 IF AND ONLY IF v1.1 includes subscriptions.

READ FIRST:
- Backlog v1.1 billing stories
- Data Model Spec v1.1: entitlement/tier storage approach
- Rules Engine Spec v1.1: tier gates (if defined)

SCOPE:
- Paywall UI
- Entitlement caching
- Tier gates enforced in engine AND UI

DO NOT:
- Implement token “pay-to-break” unless v1.1 explicitly includes it.
- Add server-side token ledger unless spec includes it.
```

---

## Epic E9 — Accountability partner (only if included in v1.1)

```text
You are an autonomous coding agent implementing Epic E9 (partner approvals) for v1.1 IF AND ONLY IF v1.1 includes partner unlock.

READ FIRST:
- Backlog v1.1 partner stories
- Rules Engine Spec v1.1: partner unlock method rules (if present)
- Data Model Spec v1.1: cloud + local mirror requirements (if present)

IMPORTANT:
- Partner unlock on iOS should NOT assume shield can open your app.
- Approval changes state; user may have to reopen blocked app or your app to proceed.
```

---

## Epic E10 — Progress & weekly review v1.1

```text
You are an autonomous coding agent implementing Epic E10 (progress insights) for v1.1.

READ FIRST:
- Backlog v1.1: progress, weekly review, hardest hour
- Data Model Spec v1.1: day_summary and unlock_attempt timestamps
- Rules Engine Spec v1.1: streak explanation

SCOPE:
- Progress dashboard from day_summary + economy_state
- Weekly review aggregation (local-only)
- Hardest hour histogram (timezone-aware)
- Streak explanation copy
- Optional local notifications (opt-in only)
```

---

## Epic E11 — iOS Screen Time enforcement v1.1 (REAL BLOCKER)

```text
You are an autonomous coding agent implementing Epic E11 (iOS Screen Time enforcement) for v1.1.

READ FIRST:
- Backlog v1.1: iOS enforcement stories (authorization, selection, shielding schedules, grant integration)
- Apple Screen Time API constraints:
  - shield cannot deep-link to the parent app reliably
  - user flow must still work without redirect
- Feature flag: ios_enforcement_enabled must gate ALL enforcement behavior.

SCOPE:
1) Capabilities/targets setup (Xcode / prebuild):
   - Add Screen Time frameworks usage:
     - FamilyControls
     - DeviceActivity
     - ManagedSettings
     - ManagedSettingsUI
   - Add required extension targets:
     - DeviceActivityMonitor extension
     - ShieldConfiguration extension
     - ShieldAction extension (if spec needs it)
   - Add App Group capability for app + all extensions
   - Ensure EAS build compatibility for dev + release

2) Authorization:
   - Provide JS-callable native module:
     - requestScreenTimeAuthorization(): {status}
     - getAuthorizationStatus()

3) App selection:
   - Present FamilyControls picker from RN (native modal)
   - Persist selections per Mode per spec (token-based storage)
   - Ensure extensions can read the selections (App Group)

4) Shield enforcement:
   - Implement baseline shielding for active Modes:
     - When a mode interval starts: apply shield to that mode’s selected apps/categories
     - When it ends: remove shield (but keep shields for other active modes)
   - If multiple active modes overlap, apply the effective mode precedence rules from Rules Engine Spec v1.1
     OR if spec defines an alternate merge strategy, follow spec exactly.

5) Unlock grants integration:
   - When the app creates an unlock grant for an app:
     - remove that app from shield for the grant duration
     - ensure shield is re-applied after grant expires
   - Must work even if main app is not running:
     - rely on DeviceActivity schedules to run extensions at boundaries
     - or the approach defined by the v1.1 spec
   - Extension code must compute “effective shield set” from persisted state:
     - active modes at now
     - active grants at now
     - precedence/merging behavior exactly per v1.1 specs

6) Custom shield UI:
   - Customize title/subtitle/buttons with your branding.
   - Show clear instruction:
     - “Open Earned to unlock” (do NOT promise redirect).
   - Use close/defer responses as supported; do not rely on launching the app.

REQUIRED ENGINE CONSISTENCY:
- The native enforcement behavior must match the same persisted mode/grant state used by JS rules engine.
- If you cannot safely reuse JS logic, replicate only what is necessary in Swift, derived from persisted data.

TESTING:
- Manual checklist required:
  - Mode schedule shields selected apps during window
  - Opening a shielded app shows custom shield
  - Creating an unlock grant from inside your app removes shield for that app
  - At grant end, shield is restored
  - Overlapping modes respect precedence rules
- Include at least one automated unit test for:
  - token/selection serialization/deserialization in App Group (if applicable)
  - shield set computation given mock state (pure Swift function)

DO NOT:
- Claim you can auto-redirect from shield into the app.
- Implement “shortcuts auto-open” unless v1.1 includes it explicitly.
```

---

## Epic E12 — Settings, privacy, compliance, quality v1.1

```text
You are an autonomous coding agent implementing Epic E12 (settings/privacy/quality) for v1.1.

READ FIRST:
- Backlog v1.1: settings, retention cleanup, diagnostics export, test suite
- Data Model Spec v1.1: user_settings JSON, retention policies, diagnostics contents
- Rules Engine Spec v1.1: event log envelope + truth-table tests

SCOPE:
- Settings screen:
  - quest toggles
  - credit costs (if allowed)
  - show Screen Time status (authorized/not)
- Retention cleanup job (event_log + attempts/grants retention)
- In-app privacy policy + terms screens
- Diagnostics export JSON (user-initiated)
- CI test suite includes rules truth-table

DO NOT:
- Upload app usage logs remotely in v1.1 unless spec explicitly says so.
```

---

## Epic E13 — Android (post-iOS; NOT in iOS-first v1.1 unless explicitly requested)

```text
Android enforcement is out of scope for iOS-first v1.1 unless the v1.1 backlog explicitly includes it.
If included later, implement behind android_enforcement_enabled flag only.
```

---

## What I recommend you do right now (so agents don’t drift)

1. **Assign E11 early** (in parallel with E1/E2), because iOS enforcement architecture (App Group storage, extensions, token serialization) affects core storage decisions.
2. Ensure the v1.1 backlog explicitly states the **unlock-from-shield behavior** (likely: “shield instructs user to open app manually”), so no agent tries to implement unsupported “redirect” hacks.

If you want, I can also generate a **branch/PR plan** that minimizes merge conflicts (especially between E1 DB-path work and E11 App Group work).

[1]: https://developer.apple.com/videos/play/wwdc2021/10123/ "Meet the Screen Time API - WWDC21 - Videos - Apple Developer"
[2]: https://developer.apple.com/forums/tags/managed-settings?page=4&sortBy=newest "Managed Settings | Apple Developer Forums"
