## Rules Engine Spec v1.1

**Status:** Code-ready, frozen behavioral truth for v1.1.
**Authority order:** Rules Engine Spec > Data Model Spec > Backlog.
**Scope emphasis:** iOS ships as a **real blocker** day 1 (Screen Time shielding). The rules engine remains the single behavioral truth for credit economy, streaks, modes, and unlock windows.

**v1.2 monetization override (applies to implementation):**
* Tiers are **Free + Pro** only.
* Credit unlock durations remain **5/15/30** in engine; UI/plan gating shows **Free = [5]**, **Pro = [5,15,30]**.
* Free cap for distracting apps = **3** (enforced in UI/validation, not schema).
* On downgrade (trial end/lapse): keep highest-priority mode active, lock other modes, enforce 3-app cap, hide 15/30, honor active grants until expiry.
* Pro purchase options: monthly + annual (trial) + lifetime (non-consumable).
* Trial/paywall triggers: 4th distracting app, 2nd mode, 15/30 unlocks, Strict/Hard enable.

---

# 1) Purpose and scope

The Rules Engine determines, at any moment:

1. **Is a target app allowed or blocked right now?**
2. If blocked, **which unlock options are available** (credits / quests / emergency)?
3. If an unlock is performed, **what state changes occur** (deduct credits, create/extend unlock window grants, enforce limits/cooldowns, update streak, log events)?

### In scope (v1.1)

* Day boundary and day IDs (04:00 local time)
* Credits accounting (earn/spend/reset/rollover)
* Streak qualification + streak bonus
* Modes (schedule + manual override + precedence)
* Blocking decision per app (logical; platform enforcement consumes it)
* Unlock windows (“grants”) + stacking/extension rule
* Quests:

    * breathing + copy text (+ QR if enabled)
    * direct unlock windows
    * per-app daily limit + cooldown
* Emergency unlock:

    * daily limit
    * delay + window
    * no same-app consecutive rule
    * **always available even in Hard mode** (safety)
* Focus sessions:

    * strict focus protection (no unlocks during focus)
    * credit awarding rules
* Anti-gaming for habit completions
* Deterministic event logging requirements for every decision/mutation

### Explicitly out of scope (v1.1)

* *How* iOS/Android enforce blocking (native module / extension implementation details)
* Billing/RevenueCat details (engine consumes “tier” only)
* Cloud sync (engine is local-first)
* Partner approvals and paid token overrides (reserved for later versions)

---

# 2) Definitions and data types

## 2.1 Identifiers

* `UserId`: string (UUID)
* `ModeId`: string (UUID)
* `FocusSessionId`: string (UUID)
* `HabitId`: string (UUID)
* `UnlockAttemptId`: string (UUID)
* `UnlockGrantId`: string (UUID)
* `QuestSessionId`: string (UUID)

## 2.2 Platform and app identity

### Platform

* `Platform = "ios" | "android"`

### Canonical AppId

The engine uses one opaque string key for apps:

* Android: `android:<packageName>`
* iOS: `iosToken:<opaqueTokenHandle>`

Examples:

* `android:com.instagram.android`
* `iosToken:AbCDeF...` (opaque, stable only on this device)

**Rules for iOS tokens:**

* Treat `iosToken:*` as **opaque** (do not parse for semantics).
* The iOS native layer is responsible for producing stable token handles for:

    * selection (FamilyControls picker)
    * shielding
    * lookup (optional display label mapping)

**Important consequence:** iOS app identifiers may not be portable across devices and should not be assumed sync-friendly.

---

# 3) Time representation and `day_id`

## 3.1 Time representation

All engine decisions must be deterministic given a time meta envelope:

`NowMeta`:

* `ts_utc_ms: number` (epoch ms)
* `timezone_id: string` (IANA zone, e.g. `"America/Vancouver"`)

Derived:

* `day_id: string` computed from `ts_utc_ms` + `timezone_id` using §3.3

## 3.2 Day boundary

A “day” starts at **04:00 local time** for the user’s timezone at the moment of evaluation.

## 3.3 `day_id` definition

`day_id` is the **local calendar date** of the day start.

Algorithm:

1. Convert `ts_utc_ms` to local date-time in `timezone_id`.
2. If local time is between `00:00:00.000` and `03:59:59.999`, subtract 1 day from the local date.
3. Return `day_id = YYYY-MM-DD` for that adjusted date.

DST / timezone travel:

* Always compute using the `timezone_id` at event time.
* Store `timezone_id` and `day_id` on every event to preserve determinism.

---

# 4) Entitlement tiers (feature gates)

The engine accepts an input tier:

`EntitlementTier = "FREE" | "PRO"`

Engine-level gates:

* `max_modes`

    * FREE: 1
    * PRO: unlimited

* `strictness_max`

    * FREE: Gentle
    * PRO: Hard

* `quests_enabled_types`

    * FREE: Breathing only
    * PRO: Breathing + Copy Text + QR (if QR key set)

* `credit_cost_customization`

    * FREE: default costs only
    * PRO: customizable within bounds (see §9.4)

> Note: Partner approvals and token overrides are intentionally not in v1.1.

---

# 5) Core state objects

## 5.1 EconomyState (singleton)

* `current_day_id: string`
* `credit_balance: number` (integer, ≥0)
* `rollover_cap: number` (default 10)
* `daily_reset_hour_local: number` (fixed 4 for v1.1)

Streak:

* `streak_count: number` (≥0)
* `streak_bonus_awarded_day_id: string | null`

Qualification flags (reset daily):

* `today_focus_qualified: boolean`
* `today_habit_qualified_count: number`
* `today_qualified: boolean`

Anti-gaming:

* `habit_awards_suspended_until_day_id: string | null`

## 5.2 EmergencyState (singleton)

* `last_used_day_id: string | null`
* `used_count_today: number`
* `last_app_id_used: AppId | null`

## 5.3 Mode

A Mode is:

* `mode_id: ModeId`

* `name: string`

* `priority: number` (higher wins)

* `strictness: "GENTLE" | "STRICT" | "HARD"`

* `schedule: TimeWindow[]`

* `blocked_apps: AppId[]`

Manual override:

* `manual_override.state: "AUTO" | "FORCED_ON" | "FORCED_OFF"`
* `manual_override.effective_at_ts_utc_ms: number | null`
* `manual_override.expires_at_ts_utc_ms: number | null`

Unlock policy (v1.1)

* Credits unlock: **always allowed** (if user has credits)
* Quests:

    * allowed in Gentle/Strict (subject to limits)
    * **not allowed in Hard** (fixed rule)
* Emergency:

    * **always allowed in all strictness tiers** (fixed rule; safety)
    * subject to daily limit and same-app rule

Settings lock policy (derived by strictness in v1.1):

* Gentle:

    * edits allowed while active
    * disable immediate
* Strict:

    * edits locked while active
    * disable cooldown 15 minutes
* Hard:

    * edits locked while active
    * cannot disable while active

## 5.4 TimeWindow

* `days_of_week: number[]` (0=Sun … 6=Sat)
* `start_local_hhmm: "HH:MM"`
* `end_local_hhmm: "HH:MM"`
* `crosses_midnight` derived:

    * `true` if end < start

Interpretation:

* Start inclusive, end exclusive: `[start, end)`

## 5.5 UnlockGrant (allow window)

An UnlockGrant permits access to one app for a time range.

* `grant_id: UnlockGrantId`

* `app_id: AppId`

* `created_ts_utc_ms: number`

* `starts_ts_utc_ms: number`

* `ends_ts_utc_ms: number`

* `method: "CREDITS" | "QUEST" | "EMERGENCY"`

* `source_mode_id: ModeId | null`

* `source_unlock_attempt_id: UnlockAttemptId`

* `metadata`:

    * Credits: `{ cost_credits, duration_minutes }`
    * Quest: `{ quest_type }`
    * Emergency: `{ delay_seconds }`

Active grant rule:

* Grant is active if `starts_ts_utc_ms <= now < ends_ts_utc_ms`

**Single-grant-per-app rule (v1.1):**

* At most one current grant per `app_id`.
* New grants **extend** the current grant (see §12).

---

# 6) Engine lifecycle

Every public entrypoint must begin by calling:

`ensureDayState(nowMeta)`

This guarantees daily rollover happens deterministically before any evaluation/mutation.

---

# 7) Daily rollover rules

## 7.1 Trigger

Rollover occurs when:

* `computed_day_id(nowMeta) != economy.current_day_id`

## 7.2 Actions (atomic)

On rollover:

1. `yesterday = economy.current_day_id`
2. `today = computed_day_id(nowMeta)`
3. Credits rollover:

    * `carry = min(economy.credit_balance, rollover_cap)` (default 10)
    * `economy.credit_balance = carry`
4. Reset daily flags:

    * `today_focus_qualified = false`
    * `today_habit_qualified_count = 0`
    * `today_qualified = false`
    * `streak_bonus_awarded_day_id = null`
    * If `habit_awards_suspended_until_day_id == yesterday`, clear it
5. Set `economy.current_day_id = today`
6. Emit `DAY_ROLLOVER` event with carried credits.

**Rule:** Streak count does not change during rollover. It updates only when the user performs a qualifying action (see §8).

---

# 8) Streak qualification and streak bonus

## 8.1 Qualifying actions

A day becomes “qualified” if either:

A) Focus qualification

* A focus session completes with:

    * `completed == true` AND
    * `duration_minutes >= 20`

B) Habit qualification

* The user completes **2 valid habit completions** in the day,
  where “valid” means:

    * completion awarded credits (not suspended)
    * completion is unique per habit per day

## 8.2 When streak updates

Streak updates once per day, when the day first becomes qualified:

If `economy.today_qualified == false` and a qualification occurs:

* `economy.today_qualified = true`
* Determine whether the previous day was qualified:

    * `yesterday_qualified = DaySummary[prev_day_id(today)].qualified == true`
* Update streak:

    * if yesterday_qualified: `streak_count += 1`
    * else: `streak_count = 1`
* Set `DaySummary[today].qualified = true`
* Set `DaySummary[today].first_qualified_ts = now`

Emit `STREAK_QUALIFIED`.

## 8.3 Streak bonus awarding

Awarded once per day, immediately after first qualification, if not already awarded for this `day_id`.

Bonus based on **new streak count**:

* 1 → 0
* 2 → +5
* 3 → +10
* 4 → +15
* 5+ → +20 (cap)

Rules:

* Bonus is **never** granted at day start.
* Bonus is only granted after the first qualifying action of the day.
* Set `economy.streak_bonus_awarded_day_id = today` after awarding.
* Emit `STREAK_BONUS_AWARDED` when bonus > 0.

---

# 9) Credits economy rules

## 9.1 Earning credits from focus sessions

`onFocusSessionCompleted(session)`

Inputs:

* `duration_minutes: number`
* `completed: boolean`

Rules:

* If `completed == false`: award 0, does not qualify.
* If `duration_minutes < 20`: award 0, does not qualify.
* Else:

    * `credits = floor(duration_minutes / 25) * 10`
    * If `duration_minutes >= 50`, add `+5` one-time bonus

Examples:

* 25 min → 10
* 50 min → 25
* 75 min → 35

After awarding:

* Update `economy.credit_balance += credits`
* Update day summary aggregates
* Emit `CREDITS_EARNED` with source `FOCUS`
* Apply qualification logic (§8)

## 9.2 Earning credits from habit completion

`onHabitCompleted(habit_id, nowMeta)`

Rules:

* A habit may be completed at most once per day.
* Default reward: +5 credits (habit-configurable within allowed bounds)

Anti-gaming:

* If > 20 habit completions occur in any rolling 60-second window:

    * set `habit_awards_suspended_until_day_id = next_day_id(today)`
    * emit `HABIT_AWARD_SUSPENDED`
* While suspended:

    * habit completions still record, but award 0 credits
    * do not count toward streak qualification

If not suspended:

* Award credits
* Increment `today_habit_qualified_count`
* Emit `CREDITS_EARNED` with source `HABIT`
* If count reaches 2, apply qualification logic (§8)

## 9.3 Spending credits (atomic)

`spendCredits(cost)`

* If `credit_balance < cost`: reject
* Else: `credit_balance -= cost`

Credit spend must be atomic with grant creation (§11.3).

## 9.4 Credit cost table (defaults + bounds)

Unlock durations:

* 5 / 15 / 30 minutes

Default costs:

* 5 min → 10 credits
* 15 min → 25 credits
* 30 min → 45 credits

Customization:

* FREE: fixed defaults
* PRO: user can customize within bounds:

    * 5 min: 5..30
    * 15 min: 15..60
    * 30 min: 30..120
    * Must be strictly increasing with duration

---

# 10) Mode activation rules

## 10.1 Schedule evaluation

A mode is “scheduled active” if `now_local` falls within any `TimeWindow`, including cross-midnight windows.

Cross-midnight interpretation:

* Window where `end < start` is treated as:

    * `[start, 24:00)` on listed day
    * `[00:00, end)` on next day

## 10.2 Manual override evaluation

* AUTO → active = scheduledActive
* FORCED_ON → active if now is within effective_at and expires_at (if set)
* FORCED_OFF → active = false while override is effective

## 10.3 Mode precedence

Given `app_id` and `now`:

1. Compute all active modes.
2. Filter to modes that include `app_id` in blocked apps.
3. If none → app is not blocked by modes.
4. Else pick effective mode:

    * highest `priority`
    * tie → higher strictness (HARD > STRICT > GENTLE)
    * tie → most recently updated mode wins

Effective mode determines:

* displayed mode name
* strictness locks
* which unlock options are enabled/disabled (quests are disabled in HARD)

---

# 11) Access evaluation API

## 11.1 Core evaluation function

`evaluateAppAccess(app_id, nowMeta) -> AccessDecision`

Returns:

* `status: "ALLOW" | "BLOCK"`
* `reason: "NO_ACTIVE_BLOCK" | "UNLOCK_GRANT_ACTIVE" | "FOCUS_SESSION_ACTIVE" | "MODE_BLOCKED"`
* `effective_mode_id: ModeId | null`
* `active_grant: UnlockGrant | null`
* `unlock_options: UnlockOption[]` (only if BLOCK)

Evaluation order (deterministic):

1. `ensureDayState(nowMeta)`
2. If a strict focus session is active AND `app_id` is in focus blocklist:

    * return BLOCK reason `FOCUS_SESSION_ACTIVE`
    * no unlock options
3. If there is an active unlock grant for app_id:

    * return ALLOW reason `UNLOCK_GRANT_ACTIVE`
4. Determine effective mode for app_id:

    * If none: return ALLOW reason `NO_ACTIVE_BLOCK`
    * Else: return BLOCK reason `MODE_BLOCKED`, compute unlock options (§11.2)

## 11.2 Unlock options computation

`UnlockOption` types (v1.1):

* `CREDITS_UNLOCK`
* `QUEST_UNLOCK`
* `EMERGENCY_UNLOCK`

Each option includes:

* `enabled: boolean`
* `disabled_reason: string | null`
* additional fields (costs, durations, cooldown)

Option availability rules:

### Credits unlock

Enabled when:

* user has enough credits for at least one duration option

Disabled reasons include:

* `INSUFFICIENT_CREDITS`

### Quest unlock

Enabled when:

* strictness is not HARD
* quest daily limit not reached for this app
* quest cooldown not active for this app
* quest type is enabled by tier and/or configured

Disabled reasons include:

* `DISALLOWED_IN_HARD_MODE`
* `DAILY_LIMIT_REACHED`
* `COOLDOWN_ACTIVE`
* `QUEST_TYPE_NOT_AVAILABLE` (e.g., Free tier)
* `QR_KEY_NOT_SET` (for QR quest)

### Emergency unlock

Enabled when:

* emergency daily limit not reached
* app_id is not the same as last emergency-used app (no consecutive same app rule)

Disabled reasons include:

* `DAILY_LIMIT_REACHED`
* `SAME_APP_CONSECUTIVE_BLOCKED`

**Hard-mode safety rule (v1.1):**

* Emergency is always evaluated and offered in HARD mode (subject to limits).
* This prevents “no options available” dead-ends.

---

# 12) Unlock system rules

## 12.1 UnlockAttempt record (always created)

When the user enters the Unlock Gate screen, create an `UnlockAttempt`:

Fields:

* `attempt_id`
* `app_id`
* `ts_utc_ms`, `timezone_id`, `day_id`
* `effective_mode_id`
* `effective_strictness`
* `available_options_snapshot` (exact options shown + reasons)
* `chosen_option` (initially null)
* `outcome: "PENDING" | "GRANTED" | "DENIED" | "CANCELLED" | "EXPIRED"`
* `method_used` (set on grant)

## 12.2 Credit unlock grant creation (atomic)

`grantCreditUnlock(attempt_id, duration_minutes ∈ {5,15,30})`

Atomic operation:

1. Validate attempt exists and is PENDING
2. Compute cost from settings/cost table
3. `spendCredits(cost)` (reject if insufficient)
4. `createOrExtendGrant(app_id, duration_minutes, method="CREDITS", metadata={cost,duration})`
5. Mark attempt outcome GRANTED, set grant_id, set method_used=CREDITS
6. Update day summary credits_spent
7. Emit:

    * `UNLOCK_OPTION_SELECTED`
    * `CREDITS_SPENT`
    * `UNLOCK_GRANTED`

## 12.3 Quest unlock rules (direct unlock)

Quests grant **a direct unlock window** (do not award credits).

Defaults:

* Unlock duration: **5 minutes**

Limits:

* Max **2 successful quest grants per app per day**
* Cooldown: **15 minutes** between successful quest grants per app
  (cooldown timer starts at the grant start time)

Eligibility (before starting quest):

* If strictness is HARD → quest disabled `DISALLOWED_IN_HARD_MODE`
* If `quest_grants_today(app_id) >= 2` → disabled `DAILY_LIMIT_REACHED`
* If `(now - last_quest_grant_start_ts) < 15 min` → disabled `COOLDOWN_ACTIVE`

Quest session model:

* `QuestSession` has:

    * `quest_type: BREATHING | COPY_TEXT | QR_SCAN`
    * `expires_ts = created_ts + 3 minutes`
    * `state: ACTIVE | COMPLETED | FAILED | EXPIRED | CANCELLED`

Success behavior (atomic on completion):

1. Create/extend grant:

    * duration 5
    * method=QUEST
    * metadata includes quest_type
2. Increment per-app quest daily counter and update last quest grant start timestamp
3. Mark attempt outcome GRANTED, method_used=QUEST
4. Emit:

    * `QUEST_COMPLETED`
    * `UNLOCK_GRANTED`

Failure behavior:

* If quest FAIL/EXPIRE/CANCEL:

    * No grant created
    * Do **not** increment daily count
    * Do **not** start cooldown
    * Attempt outcome is DENIED (failed/expired) or CANCELLED (cancel)
    * Emit `QUEST_FAILED` or `QUEST_CANCELLED` (see §13)

## 12.4 Emergency unlock rules

Emergency unlock is free but constrained.

Defaults:

* Daily limit: **1** per day (global)
* Delay: **60 seconds** before unlock begins
* Unlock duration: **5 minutes**
* Cannot use emergency for the same app twice consecutively

Eligibility:

* If `used_count_today >= 1` → disabled `DAILY_LIMIT_REACHED`
* If `last_app_id_used == app_id` → disabled `SAME_APP_CONSECUTIVE_BLOCKED`

Grant creation (atomic):

1. Create/extend grant with delayed start:

    * `starts = now + 60s`
    * `ends = starts + 5min`
    * method=EMERGENCY
2. Update emergency state:

    * `used_count_today += 1`
    * `last_used_day_id = today`
    * `last_app_id_used = app_id`
3. Mark attempt outcome GRANTED, method_used=EMERGENCY
4. Emit:

    * `UNLOCK_OPTION_SELECTED`
    * `EMERGENCY_USED`
    * `UNLOCK_GRANTED`

---

# 13) Grant creation, extension, and stacking

## 13.1 Single-grant-per-app invariant

Maintain at most one “current” grant per `app_id`.

A grant can be:

* active now, or
* pending (starts in the future, e.g. emergency delay)

## 13.2 `createOrExtendGrant(app_id, duration_minutes, method, metadata)`

If no current grant exists:

* Create new grant:

    * starts = now (except emergency delayed)
    * ends = starts + duration
    * store method/metadata

If a current grant exists:

* Extension rule:

    * `base = max(existing.ends_ts, now)`
    * `new_end = base + duration_minutes`
    * Update existing grant `ends_ts = new_end`
    * Preserve original starts_ts
    * Log an extension record (optional) OR log via event payload

**Important:** Repeated unlocks extend time; they do not reset the start time.

---

# 14) Focus sessions (strict focus protection)

## 14.1 FocusState

* `active: boolean`
* `session_id`
* `started_ts_utc_ms`
* `planned_duration_minutes`
* `blocklist_app_ids: AppId[]`
* `strict = true` (v1.1 fixed)

## 14.2 Focus blocking rule

If focus session is active and `app_id` is in focus blocklist:

* Always BLOCK, reason `FOCUS_SESSION_ACTIVE`
* No unlock methods allowed during focus
* Ending focus early results in:

    * completed=false
    * 0 credits
    * no qualification

---

# 15) Strictness protections (engine-enforced)

These protections are enforced both in UI and in engine-level mutators.

## 15.1 Editing a mode while active

* Gentle: allowed
* Strict/Hard: disallow changes to:

    * schedule
    * blocked apps list
    * strictness
    * priority
    * quest availability for that mode (if you expose it)

Allowed changes:

* name/icon (non-functional)

## 15.2 Disabling an active mode

If user requests “disable now”:

* Gentle:

    * apply FORCED_OFF immediately

* Strict:

    * apply a pending disable:

        * effective_at = now + 15 minutes
    * mode remains active until effective_at

* Hard:

    * deny: `CANNOT_DISABLE_HARD_MODE_WHILE_ACTIVE`

---

# 16) iOS enforcement integration notes (non-engine, but required for correctness)

Because iOS is shipping as a real blocker from day 1, the native layer must apply the engine’s decisions.

## 16.1 Key constraint

The iOS shield cannot open/redirect into the main app.
Therefore, unlocks happen **in-app**, not “inside the shield.”

## 16.2 lastBlockedAppId handoff contract

* The iOS extension writes `lastBlockedAppId: AppId` (an `iosToken:*` value) to App Group storage when it shows a shield for a blocked app.
* When the user opens Earned, the app reads `lastBlockedAppId` and routes to:

    * `UnlockGate(lastBlockedAppId)`
* The unlock attempt created uses this `app_id`.

## 16.3 Applying unlock windows

When an UnlockGrant becomes active for an iOS app token:

* native layer must remove/relax shielding for that token during `[starts, ends)`
* then re-apply shielding when the window ends

The engine remains the source of truth for:

* grant start/end times
* extension rules
* limits/cooldowns

---

# 17) Logging requirements (minimum event set)

Every state mutation must emit an append-only event. Event envelopes include:

* `event_id`
* `ts_utc_ms`
* `timezone_id`
* `day_id`
* `type`
* `payload`

Required event types (v1.1):

Daily / economy:

* `DAY_ROLLOVER` (from_day_id, to_day_id, carried_credits)
* `CREDITS_EARNED` (source: FOCUS/HABIT/STREAK_BONUS, amount, ref ids)
* `CREDITS_SPENT` (app_id, amount, minutes, attempt_id)
* `STREAK_QUALIFIED` (day_id, method, new_streak_count, first_qualified_ts)
* `STREAK_BONUS_AWARDED` (day_id, amount, streak_count)
* `HABIT_AWARD_SUSPENDED` (until_day_id, reason)

Unlock flow:

* `UNLOCK_ATTEMPT_CREATED` (attempt_id, app_id, mode_id, strictness)
* `UNLOCK_OPTION_SELECTED` (attempt_id, option_type, payload)
* `UNLOCK_GRANTED` (grant_id, app_id, method, starts, ends, attempt_id)
* `UNLOCK_DENIED` (attempt_id, reason)
* `UNLOCK_CANCELLED` (attempt_id)

Quest flow:

* `QUEST_STARTED` (quest_session_id, attempt_id, quest_type)
* `QUEST_COMPLETED` (quest_session_id, attempt_id, quest_type)
* `QUEST_FAILED` (quest_session_id, attempt_id, quest_type, reason)
* `QUEST_CANCELLED` (quest_session_id, attempt_id, quest_type)
* `QUEST_EXPIRED` (quest_session_id, attempt_id, quest_type)

Emergency:

* `EMERGENCY_USED` (attempt_id, app_id, delay_seconds, unlock_minutes)

Focus:

* `FOCUS_STARTED` (session_id, planned_minutes)
* `FOCUS_ENDED` (session_id, completed, actual_minutes, ended_early)
* `FOCUS_CREDITS_AWARDED` (session_id, amount)

> Note: The exact payload schemas will be specified in the Data Model Spec v1.1 and must match.

---

# 18) Deterministic test scenarios (must implement)

These “truth-table” scenarios must be covered by automated tests.

## 18.1 Day boundary and rollover

* 03:59 local → day_id = yesterday
* 04:00 local → day_id = today
* Rollover carries min(10, balance)
* Rollover resets daily flags

## 18.2 Streak qualification and bonus

* Day becomes qualified only on first qualifying action
* Bonus awarded once/day after first qualifying action
* Missing a day resets streak to 1 on next qualification

## 18.3 Habit anti-gaming

* > 20 completions in 60s triggers suspension until next day
* Suspended completions award 0 and don’t qualify streak
* Focus qualification still works during suspension

## 18.4 Quest limits and cooldown

* Two successful quest grants for same app same day: allowed
* Third: disabled `DAILY_LIMIT_REACHED`
* Cooldown: second quest attempt within 15 min after last successful quest grant: disabled `COOLDOWN_ACTIVE`
* Quest failures do not consume limit or start cooldown

## 18.5 Grant extension behavior

* Credits unlock 5 min, then credits unlock 5 min again before expiry:

    * same grant exists
    * end time extends by +5 from previous end
* Emergency grant delayed start extends correctly if another unlock happens during delay

## 18.6 Emergency rules

* Emergency used once/day → allowed
* Second emergency same day → disabled `DAILY_LIMIT_REACHED`
* Emergency cannot be used for same app twice consecutively
* Emergency remains available in Hard mode (subject to limits)

## 18.7 Focus blocking

* While focus session active, blocked apps return BLOCK reason `FOCUS_SESSION_ACTIVE`
* No unlock options presented
* Ending early awards 0 credits and no qualification

---
