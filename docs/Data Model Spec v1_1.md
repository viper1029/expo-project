## Data Model Spec v1.1

**Status:** Code-ready storage truth for v1.1.
**Authority order:** Rules Engine Spec v1.1 > Data Model Spec v1.1 > Backlog.
**Core update from v1.0:** v1.1 is **iOS-first real blocking** and **does NOT include Partner or Token systems yet**. The data model is simplified accordingly, and iOS app identities use **opaque Screen Time tokens** (`iosToken:*`) rather than bundle IDs.

---

# 0) Frozen storage decisions (v1.1)

## 0.1 Local-first source of truth

**SQLite is authoritative** for:

* Modes + schedules + blocked apps
* Global “Distracting Apps” list
* Credits / streak state
* Focus sessions + habit completions
* Unlock attempts, grants, quest sessions
* Emergency state
* Event log (append-only)

**No cloud sync is required in v1.1.**
Rationale: iOS app selection uses opaque tokens that aren’t portable across devices, and we’re prioritizing iOS-first launch speed and correctness.

## 0.2 iOS App Group requirement (real blocker day 1)

On iOS, the app and its Screen Time extensions must share data. Therefore:

* The SQLite database file **MUST live in the App Group shared container** so extensions can read it if needed for UI (mode name, etc.) and to ensure consistent state visibility.

> Extensions should avoid writing heavy state into SQLite unless absolutely necessary (reads are fine). See §0.3.

## 0.3 iOS “last blocked app” handoff uses App Group Shared KV

To make iOS unlocking smooth (user opens Earned → auto-routes to the right Unlock Gate), the iOS shield writes ephemeral keys to **App Group Shared UserDefaults** (or equivalent shared KV), not SQLite.

Required keys:

* `ios_last_blocked_app_id` → string (AppId, e.g. `iosToken:...`)
* `ios_last_blocked_ts_utc_ms` → string integer (epoch ms)
* `ios_last_blocked_mode_id` → string (optional; ModeId if known)

These keys are **ephemeral** UX helpers; they are not part of the rules engine truth and do not affect credit/grant logic.

---

# Part A — Local Database (SQLite)

## A1) SQLite configuration requirements (must-do)

At app start (every launch):

* `PRAGMA foreign_keys = ON;`
* `PRAGMA journal_mode = WAL;`
* `PRAGMA synchronous = NORMAL;` (or FULL for max safety)

All correctness-critical mutations **must** run inside explicit transactions.

---

## A2) Canonical enums (strings must match exactly)

### Strictness

* `GENTLE`, `STRICT`, `HARD`

### Manual override state

* `AUTO`, `FORCED_ON`, `FORCED_OFF`

### Unlock attempt outcome

* `PENDING`, `GRANTED`, `DENIED`, `CANCELLED`, `EXPIRED`

### Unlock grant method

* `CREDITS`, `QUEST`, `EMERGENCY`

### Quest type

* `BREATHING`, `COPY_TEXT`, `QR_SCAN`

### Quest session state

* `ACTIVE`, `COMPLETED`, `FAILED`, `EXPIRED`, `CANCELLED`

### Entitlement tier

* `FREE`, `PLUS`, `PRO`

### Sync status (kept for future-proofing; v1.1 is local-only)

* `LOCAL_ONLY`, `PENDING`, `SYNCED`, `FAILED`

---

## A3) SQLite schema (DDL)

> Notes:
>
> * All timestamps are `*_ts_utc_ms` integers (epoch ms).
> * JSON stored as TEXT.
> * Soft delete (tombstones) used for Modes/Habits to preserve auditability and future sync compatibility.

```sql
-- =========================
-- META / KEY-VALUE
-- =========================

CREATE TABLE IF NOT EXISTS meta (
  key TEXT PRIMARY KEY,
  value TEXT NOT NULL
);

-- required meta keys:
-- - schema_version: "1"
-- - device_id: "<uuid>"
-- - db_location: "app_sandbox" | "app_group" (iOS should be "app_group")
-- - last_retention_cleanup_ts_utc_ms: "<int>" (optional)
-- - flags_json: "{...}" (optional; feature flags can live here)

-- =========================
-- ENTITLEMENTS (singleton)
-- =========================
CREATE TABLE IF NOT EXISTS entitlement_state (
  id INTEGER PRIMARY KEY CHECK (id = 1),

  tier TEXT NOT NULL DEFAULT 'FREE',                 -- FREE|PLUS|PRO
  subscription_active INTEGER NOT NULL DEFAULT 0 CHECK (subscription_active IN (0,1)),
  updated_ts_utc_ms INTEGER NOT NULL
);

-- =========================
-- USER SETTINGS (singleton)
-- =========================
CREATE TABLE IF NOT EXISTS user_settings (
  id INTEGER PRIMARY KEY CHECK (id = 1),

  settings_json TEXT NOT NULL,
  updated_ts_utc_ms INTEGER NOT NULL,
  sync_status TEXT NOT NULL DEFAULT 'LOCAL_ONLY'
);

-- =========================
-- ECONOMY STATE (singleton)
-- =========================
CREATE TABLE IF NOT EXISTS economy_state (
  id INTEGER PRIMARY KEY CHECK (id = 1),

  current_day_id TEXT NOT NULL,                      -- YYYY-MM-DD (04:00 boundary)
  credit_balance INTEGER NOT NULL DEFAULT 0 CHECK (credit_balance >= 0),

  rollover_cap INTEGER NOT NULL DEFAULT 10 CHECK (rollover_cap >= 0),
  daily_reset_hour_local INTEGER NOT NULL DEFAULT 4 CHECK (daily_reset_hour_local BETWEEN 0 AND 23),

  streak_count INTEGER NOT NULL DEFAULT 0 CHECK (streak_count >= 0),
  streak_bonus_awarded_day_id TEXT,                  -- null or YYYY-MM-DD

  today_focus_qualified INTEGER NOT NULL DEFAULT 0 CHECK (today_focus_qualified IN (0,1)),
  today_habit_qualified_count INTEGER NOT NULL DEFAULT 0 CHECK (today_habit_qualified_count >= 0),
  today_qualified INTEGER NOT NULL DEFAULT 0 CHECK (today_qualified IN (0,1)),

  habit_awards_suspended_until_day_id TEXT,          -- null or YYYY-MM-DD

  updated_ts_utc_ms INTEGER NOT NULL
);

-- =========================
-- EMERGENCY STATE (singleton)
-- =========================
CREATE TABLE IF NOT EXISTS emergency_state (
  id INTEGER PRIMARY KEY CHECK (id = 1),

  last_used_day_id TEXT,
  used_count_today INTEGER NOT NULL DEFAULT 0 CHECK (used_count_today >= 0),
  last_app_id_used TEXT,                             -- AppId string

  updated_ts_utc_ms INTEGER NOT NULL
);

-- =========================
-- APP INFO + DISTRACTING APPS (global list)
-- =========================

CREATE TABLE IF NOT EXISTS app_info (
  app_id TEXT PRIMARY KEY,                           -- android:... or iosToken:...
  platform TEXT NOT NULL CHECK (platform IN ('ios','android')),

  display_name TEXT,                                 -- best-effort label
  user_alias TEXT,                                   -- optional user override label
  icon_uri TEXT,                                     -- Android only (optional)

  created_ts_utc_ms INTEGER NOT NULL,
  updated_ts_utc_ms INTEGER NOT NULL
);

CREATE INDEX IF NOT EXISTS idx_app_info_platform ON app_info(platform);

-- Global list used as the user’s "Distracting Apps"
CREATE TABLE IF NOT EXISTS distracting_app (
  app_id TEXT PRIMARY KEY,
  created_ts_utc_ms INTEGER NOT NULL
  -- intentionally no FK to app_info to avoid accidental cascades if app_info is pruned
);

-- =========================
-- DAY SUMMARY (aggregated; used for progress/weekly review)
-- =========================
CREATE TABLE IF NOT EXISTS day_summary (
  day_id TEXT PRIMARY KEY,                           -- YYYY-MM-DD
  timezone_id TEXT NOT NULL,

  qualified INTEGER NOT NULL DEFAULT 0 CHECK (qualified IN (0,1)),
  first_qualified_ts_utc_ms INTEGER,

  focus_minutes_completed INTEGER NOT NULL DEFAULT 0,
  credits_earned INTEGER NOT NULL DEFAULT 0,
  credits_spent INTEGER NOT NULL DEFAULT 0,

  unlock_attempts_count INTEGER NOT NULL DEFAULT 0,

  credits_unlocks_count INTEGER NOT NULL DEFAULT 0,
  quest_unlocks_count INTEGER NOT NULL DEFAULT 0,
  emergency_unlocks_count INTEGER NOT NULL DEFAULT 0,

  created_ts_utc_ms INTEGER NOT NULL,
  updated_ts_utc_ms INTEGER NOT NULL
);

-- =========================
-- MODES
-- =========================
CREATE TABLE IF NOT EXISTS mode (
  mode_id TEXT PRIMARY KEY,

  name TEXT NOT NULL,
  priority INTEGER NOT NULL DEFAULT 0,

  strictness TEXT NOT NULL,                          -- GENTLE|STRICT|HARD

  manual_state TEXT NOT NULL DEFAULT 'AUTO',          -- AUTO|FORCED_ON|FORCED_OFF
  manual_effective_at_ts_utc_ms INTEGER,
  manual_expires_at_ts_utc_ms INTEGER,

  created_ts_utc_ms INTEGER NOT NULL,
  updated_ts_utc_ms INTEGER NOT NULL,
  deleted_ts_utc_ms INTEGER,                         -- tombstone

  sync_status TEXT NOT NULL DEFAULT 'LOCAL_ONLY'
);

CREATE INDEX IF NOT EXISTS idx_mode_updated ON mode(updated_ts_utc_ms);
CREATE INDEX IF NOT EXISTS idx_mode_deleted ON mode(deleted_ts_utc_ms);

CREATE TABLE IF NOT EXISTS mode_window (
  window_id TEXT PRIMARY KEY,

  mode_id TEXT NOT NULL,
  days_of_week_mask INTEGER NOT NULL CHECK (days_of_week_mask BETWEEN 0 AND 127),
  start_local_hhmm TEXT NOT NULL,                     -- "HH:MM"
  end_local_hhmm TEXT NOT NULL,                       -- "HH:MM"

  created_ts_utc_ms INTEGER NOT NULL,
  updated_ts_utc_ms INTEGER NOT NULL,

  FOREIGN KEY (mode_id) REFERENCES mode(mode_id) ON DELETE CASCADE
);

CREATE INDEX IF NOT EXISTS idx_mode_window_mode ON mode_window(mode_id);

CREATE TABLE IF NOT EXISTS mode_blocked_app (
  mode_id TEXT NOT NULL,
  app_id TEXT NOT NULL,                               -- AppId

  created_ts_utc_ms INTEGER NOT NULL,

  PRIMARY KEY (mode_id, app_id),
  FOREIGN KEY (mode_id) REFERENCES mode(mode_id) ON DELETE CASCADE
);

CREATE INDEX IF NOT EXISTS idx_mode_blocked_app_app ON mode_blocked_app(app_id);

-- =========================
-- HABITS
-- =========================
CREATE TABLE IF NOT EXISTS habit (
  habit_id TEXT PRIMARY KEY,

  name TEXT NOT NULL,
  reward_credits INTEGER NOT NULL DEFAULT 5 CHECK (reward_credits BETWEEN 0 AND 100),

  active INTEGER NOT NULL DEFAULT 1 CHECK (active IN (0,1)),
  sort_order INTEGER NOT NULL DEFAULT 0,

  created_ts_utc_ms INTEGER NOT NULL,
  updated_ts_utc_ms INTEGER NOT NULL,
  deleted_ts_utc_ms INTEGER,                          -- tombstone

  sync_status TEXT NOT NULL DEFAULT 'LOCAL_ONLY'
);

CREATE INDEX IF NOT EXISTS idx_habit_updated ON habit(updated_ts_utc_ms);

CREATE TABLE IF NOT EXISTS habit_completion (
  completion_id TEXT PRIMARY KEY,

  habit_id TEXT NOT NULL,
  ts_utc_ms INTEGER NOT NULL,
  timezone_id TEXT NOT NULL,
  day_id TEXT NOT NULL,

  awarded_credits INTEGER NOT NULL DEFAULT 0,
  award_suspended INTEGER NOT NULL DEFAULT 0 CHECK (award_suspended IN (0,1)),

  created_ts_utc_ms INTEGER NOT NULL,

  FOREIGN KEY (habit_id) REFERENCES habit(habit_id) ON DELETE RESTRICT
);

-- Enforce at most one completion per habit per day:
CREATE UNIQUE INDEX IF NOT EXISTS uq_habit_completion_per_day ON habit_completion(habit_id, day_id);
CREATE INDEX IF NOT EXISTS idx_habit_completion_day ON habit_completion(day_id);
CREATE INDEX IF NOT EXISTS idx_habit_completion_ts ON habit_completion(ts_utc_ms);

-- =========================
-- FOCUS SESSIONS
-- =========================
CREATE TABLE IF NOT EXISTS focus_session (
  session_id TEXT PRIMARY KEY,

  started_ts_utc_ms INTEGER NOT NULL,
  ended_ts_utc_ms INTEGER,

  timezone_id TEXT NOT NULL,
  day_id TEXT NOT NULL,

  planned_duration_minutes INTEGER NOT NULL CHECK (planned_duration_minutes > 0),
  actual_duration_minutes INTEGER,

  completed INTEGER NOT NULL DEFAULT 0 CHECK (completed IN (0,1)),
  ended_early INTEGER NOT NULL DEFAULT 0 CHECK (ended_early IN (0,1)),

  strict INTEGER NOT NULL DEFAULT 1 CHECK (strict IN (0,1)),
  task_label TEXT,

  awarded_credits INTEGER NOT NULL DEFAULT 0,
  qualifies INTEGER NOT NULL DEFAULT 0 CHECK (qualifies IN (0,1)),

  created_ts_utc_ms INTEGER NOT NULL,
  updated_ts_utc_ms INTEGER NOT NULL
);

CREATE INDEX IF NOT EXISTS idx_focus_session_day ON focus_session(day_id);
CREATE INDEX IF NOT EXISTS idx_focus_session_started ON focus_session(started_ts_utc_ms);

CREATE TABLE IF NOT EXISTS focus_session_blocked_app (
  session_id TEXT NOT NULL,
  app_id TEXT NOT NULL,
  PRIMARY KEY (session_id, app_id),
  FOREIGN KEY (session_id) REFERENCES focus_session(session_id) ON DELETE CASCADE
);

-- Singleton pointer for active focus session
CREATE TABLE IF NOT EXISTS focus_state (
  id INTEGER PRIMARY KEY CHECK (id = 1),

  active_session_id TEXT,
  updated_ts_utc_ms INTEGER NOT NULL,

  FOREIGN KEY (active_session_id) REFERENCES focus_session(session_id) ON DELETE SET NULL
);

-- =========================
-- UNLOCK ATTEMPTS / GRANTS
-- =========================
CREATE TABLE IF NOT EXISTS unlock_attempt (
  attempt_id TEXT PRIMARY KEY,

  app_id TEXT NOT NULL,
  ts_utc_ms INTEGER NOT NULL,
  timezone_id TEXT NOT NULL,
  day_id TEXT NOT NULL,

  effective_mode_id TEXT,
  effective_strictness TEXT NOT NULL,                 -- GENTLE|STRICT|HARD

  available_options_json TEXT NOT NULL,               -- snapshot (v1.1 schema in §A4.2)

  chosen_option_type TEXT,                            -- CREDITS_UNLOCK|QUEST_UNLOCK|EMERGENCY_UNLOCK
  chosen_option_payload_json TEXT,                    -- details (duration, quest type, etc.)

  outcome TEXT NOT NULL DEFAULT 'PENDING',            -- PENDING|GRANTED|DENIED|CANCELLED|EXPIRED
  denial_reason TEXT,

  grant_id TEXT,                                      -- set if granted
  created_ts_utc_ms INTEGER NOT NULL,
  updated_ts_utc_ms INTEGER NOT NULL,

  FOREIGN KEY (effective_mode_id) REFERENCES mode(mode_id) ON DELETE SET NULL
);

CREATE INDEX IF NOT EXISTS idx_unlock_attempt_day ON unlock_attempt(day_id);
CREATE INDEX IF NOT EXISTS idx_unlock_attempt_app ON unlock_attempt(app_id);
CREATE INDEX IF NOT EXISTS idx_unlock_attempt_ts ON unlock_attempt(ts_utc_ms);

CREATE TABLE IF NOT EXISTS unlock_grant (
  grant_id TEXT PRIMARY KEY,

  app_id TEXT NOT NULL,

  created_ts_utc_ms INTEGER NOT NULL,
  created_day_id TEXT NOT NULL,

  starts_ts_utc_ms INTEGER NOT NULL,
  ends_ts_utc_ms INTEGER NOT NULL,

  method TEXT NOT NULL,                               -- CREDITS|QUEST|EMERGENCY
  source_mode_id TEXT,
  source_attempt_id TEXT NOT NULL,

  metadata_json TEXT NOT NULL DEFAULT '{}',
  revoked_ts_utc_ms INTEGER,

  FOREIGN KEY (source_mode_id) REFERENCES mode(mode_id) ON DELETE SET NULL,
  FOREIGN KEY (source_attempt_id) REFERENCES unlock_attempt(attempt_id) ON DELETE CASCADE
);

CREATE INDEX IF NOT EXISTS idx_unlock_grant_app ON unlock_grant(app_id);
CREATE INDEX IF NOT EXISTS idx_unlock_grant_end ON unlock_grant(ends_ts_utc_ms);

-- Enforce "single current grant per app" via mapping table.
CREATE TABLE IF NOT EXISTS unlock_grant_current (
  app_id TEXT PRIMARY KEY,
  grant_id TEXT NOT NULL,
  updated_ts_utc_ms INTEGER NOT NULL,
  FOREIGN KEY (grant_id) REFERENCES unlock_grant(grant_id) ON DELETE CASCADE
);

-- Optional: store extensions explicitly (recommended for debugging)
CREATE TABLE IF NOT EXISTS unlock_grant_extension (
  extension_id TEXT PRIMARY KEY,
  grant_id TEXT NOT NULL,
  ts_utc_ms INTEGER NOT NULL,
  added_minutes INTEGER NOT NULL CHECK (added_minutes > 0),
  new_ends_ts_utc_ms INTEGER NOT NULL,
  method TEXT NOT NULL,
  metadata_json TEXT NOT NULL DEFAULT '{}',
  FOREIGN KEY (grant_id) REFERENCES unlock_grant(grant_id) ON DELETE CASCADE
);

CREATE INDEX IF NOT EXISTS idx_unlock_grant_extension_grant ON unlock_grant_extension(grant_id);

-- =========================
-- QUESTS
-- =========================
CREATE TABLE IF NOT EXISTS quest_session (
  quest_session_id TEXT PRIMARY KEY,

  attempt_id TEXT NOT NULL,
  quest_type TEXT NOT NULL,                           -- BREATHING|COPY_TEXT|QR_SCAN

  created_ts_utc_ms INTEGER NOT NULL,
  expires_ts_utc_ms INTEGER NOT NULL,

  state TEXT NOT NULL DEFAULT 'ACTIVE',               -- ACTIVE|COMPLETED|FAILED|EXPIRED|CANCELLED
  metadata_json TEXT NOT NULL DEFAULT '{}',

  updated_ts_utc_ms INTEGER NOT NULL,

  FOREIGN KEY (attempt_id) REFERENCES unlock_attempt(attempt_id) ON DELETE CASCADE
);

CREATE INDEX IF NOT EXISTS idx_quest_session_attempt ON quest_session(attempt_id);

-- Fast availability checks for quest daily limits/cooldowns
CREATE TABLE IF NOT EXISTS daily_app_limits (
  day_id TEXT NOT NULL,
  app_id TEXT NOT NULL,

  quest_grants_count INTEGER NOT NULL DEFAULT 0 CHECK (quest_grants_count >= 0),
  last_quest_grant_start_ts_utc_ms INTEGER,

  PRIMARY KEY (day_id, app_id)
);

CREATE INDEX IF NOT EXISTS idx_daily_app_limits_app ON daily_app_limits(app_id);

-- =========================
-- EVENT LOG (append-only)
-- =========================
CREATE TABLE IF NOT EXISTS event_log (
  event_id TEXT PRIMARY KEY,

  ts_utc_ms INTEGER NOT NULL,
  timezone_id TEXT NOT NULL,
  day_id TEXT NOT NULL,

  type TEXT NOT NULL,
  payload_json TEXT NOT NULL,

  created_ts_utc_ms INTEGER NOT NULL,

  sync_status TEXT NOT NULL DEFAULT 'LOCAL_ONLY'
);

CREATE INDEX IF NOT EXISTS idx_event_day ON event_log(day_id);
CREATE INDEX IF NOT EXISTS idx_event_ts ON event_log(ts_utc_ms);
CREATE INDEX IF NOT EXISTS idx_event_type ON event_log(type);
```

---

## A4) Local JSON schemas (v1.1)

### A4.1 `user_settings.settings_json` schema (v1.1)

```json
{
  "timezone_id": "America/Vancouver",

  "credit_costs": {
    "minutes_5": 10,
    "minutes_15": 25,
    "minutes_30": 45
  },

  "quest": {
    "enabled_types": ["BREATHING", "COPY_TEXT", "QR_SCAN"],
    "unlock_minutes": 5,
    "max_quest_unlocks_per_app_per_day": 2,
    "cooldown_minutes_per_app": 15,

    "qr_key_value": null,
    "qr_key_set_ts_utc_ms": null,
    "key_change_cooldown_hours_strict": 24
  },

  "emergency": {
    "daily_limit": 1,
    "delay_seconds": 60,
    "unlock_minutes": 5,
    "block_same_app_consecutive": true
  },

  "focus": {
    "preset_minutes": [25, 50, 75],
    "strict": true,
    "min_minutes_to_qualify": 20
  }
}
```

**Constraints (engine enforced):**

* credit costs must be strictly increasing
* quest unlock_minutes fixed to 5 in v1.1
* quest max/cooldown fixed to 2 and 15 in v1.1
* emergency settings fixed in v1.1 (read-only UI allowed; engine uses constants)

### A4.2 `unlock_attempt.available_options_json` snapshot schema (v1.1)

```json
{
  "generated_ts_utc_ms": 1738440000000,
  "mode_id": "mode_123",
  "mode_strictness": "HARD",
  "app_id": "iosToken:AbCDeF...",
  "options": [
    {
      "type": "CREDITS_UNLOCK",
      "enabled": true,
      "disabled_reason": null,
      "durations": [
        { "minutes": 5, "cost_credits": 10, "enabled": true, "disabled_reason": null },
        { "minutes": 15, "cost_credits": 25, "enabled": false, "disabled_reason": "INSUFFICIENT_CREDITS" },
        { "minutes": 30, "cost_credits": 45, "enabled": false, "disabled_reason": "INSUFFICIENT_CREDITS" }
      ]
    },
    {
      "type": "QUEST_UNLOCK",
      "enabled": false,
      "disabled_reason": "DISALLOWED_IN_HARD_MODE",
      "quest_types": [
        { "quest_type": "BREATHING", "enabled": false, "disabled_reason": "DISALLOWED_IN_HARD_MODE" },
        { "quest_type": "COPY_TEXT", "enabled": false, "disabled_reason": "DISALLOWED_IN_HARD_MODE" },
        { "quest_type": "QR_SCAN", "enabled": false, "disabled_reason": "DISALLOWED_IN_HARD_MODE" }
      ]
    },
    {
      "type": "EMERGENCY_UNLOCK",
      "enabled": true,
      "disabled_reason": null,
      "delay_seconds": 60,
      "unlock_minutes": 5
    }
  ]
}
```

---

## A5) Invariants (must always hold)

### A5.1 Singleton rows exist

On first launch, ensure these exist:

* `economy_state(id=1)`
* `user_settings(id=1)`
* `entitlement_state(id=1)`
* `emergency_state(id=1)`
* `focus_state(id=1)`

### A5.2 Single current grant per app

For any `app_id`, at most one row exists in `unlock_grant_current`.

### A5.3 Attempt ↔ grant consistency

If `unlock_attempt.outcome == 'GRANTED'` then:

* `unlock_attempt.grant_id IS NOT NULL`
* referenced `unlock_grant(grant_id)` exists

### A5.4 Habit completion uniqueness

Unique index ensures:

* at most one `habit_completion` per `(habit_id, day_id)`

### A5.5 AppId format constraints (logical)

* Android AppId must start with `android:`
* iOS AppId must start with `iosToken:`
  This is enforced at the domain layer (not via SQL constraint).

---

## A6) Transaction boundaries (atomic operations)

Any engine action that changes:

* economy (credits/streak)
* grants
* attempts/outcomes
* daily limits
* day_summary
* event_log
  …must occur in **one SQLite transaction**.

### Required atomic operations

#### 1) Daily rollover (`ensureDayState`)

Transaction must:

* update `economy_state` (carry credits, reset flags)
* upsert `day_summary` for new day_id
* insert `event_log` type `DAY_ROLLOVER`

#### 2) Create unlock attempt (opening Unlock Gate)

Transaction must:

* insert `unlock_attempt`
* increment `day_summary.unlock_attempts_count`
* insert `event_log` type `UNLOCK_ATTEMPT_CREATED`

#### 3) Credits unlock grant

Transaction must:

* subtract credits (`economy_state`)
* update `day_summary.credits_spent`
* create or extend current grant (`unlock_grant` + `unlock_grant_current` + `unlock_grant_extension`)
* mark attempt outcome GRANTED with grant_id
* increment `day_summary.credits_unlocks_count`
* insert events:

    * `UNLOCK_OPTION_SELECTED`
    * `CREDITS_SPENT`
    * `UNLOCK_GRANTED`

#### 4) Quest completion grant

Transaction must:

* update quest_session state COMPLETED
* create/extend grant (method QUEST)
* update `daily_app_limits` count + last start ts
* increment `day_summary.quest_unlocks_count`
* mark attempt outcome GRANTED
* insert events:

    * `QUEST_COMPLETED`
    * `UNLOCK_GRANTED`

**Quest failure/cancel/expire must NOT increment daily_app_limits.**

#### 5) Emergency unlock grant

Transaction must:

* update `emergency_state` (counts + last app)
* create delayed grant (starts in future)
* mark attempt outcome GRANTED
* increment `day_summary.emergency_unlocks_count`
* insert events:

    * `UNLOCK_OPTION_SELECTED`
    * `EMERGENCY_USED`
    * `UNLOCK_GRANTED`

#### 6) Focus session completion

Transaction must:

* update `focus_session` row (completed/ended early, durations)
* clear `focus_state.active_session_id`
* update credits and streak logic through economy state
* update `day_summary.focus_minutes_completed` and `credits_earned`
* insert events:

    * `FOCUS_ENDED`
    * `CREDITS_EARNED` (if any)
    * `STREAK_QUALIFIED` / `STREAK_BONUS_AWARDED` (if applicable)

#### 7) Habit completion

Transaction must:

* insert habit_completion (or fail on unique constraint)
* if not suspended:

    * update economy credits
    * update `today_habit_qualified_count`
    * update day_summary credits_earned
    * emit `CREDITS_EARNED`
    * if day qualifies for streak, emit `STREAK_QUALIFIED` and maybe `STREAK_BONUS_AWARDED`
* if suspension triggers:

    * set `habit_awards_suspended_until_day_id`
    * emit `HABIT_AWARD_SUSPENDED`

---

## A7) Event payload schemas (v1.1 minimum)

Each `event_log.payload_json` must match these patterns.

### `DAY_ROLLOVER`

```json
{ "from_day_id": "2026-02-01", "to_day_id": "2026-02-02", "carried_credits": 10 }
```

### `CREDITS_EARNED`

```json
{ "source": "FOCUS|HABIT|STREAK_BONUS", "amount": 10, "focus_session_id": null, "habit_id": null }
```

### `CREDITS_SPENT`

```json
{ "app_id": "android:com.instagram.android", "amount": 10, "unlock_minutes": 5, "attempt_id": "..." }
```

### `STREAK_QUALIFIED`

```json
{ "day_id": "2026-02-02", "method": "FOCUS|HABITS", "streak_count_new": 3, "first_qualified_ts_utc_ms": 1738460000000 }
```

### `STREAK_BONUS_AWARDED`

```json
{ "day_id": "2026-02-02", "amount": 10, "streak_count": 3 }
```

### `HABIT_AWARD_SUSPENDED`

```json
{ "until_day_id": "2026-02-03", "reason": "TOO_MANY_COMPLETIONS_IN_WINDOW" }
```

### `UNLOCK_ATTEMPT_CREATED`

```json
{ "attempt_id": "...", "app_id": "...", "mode_id": "...", "strictness": "STRICT" }
```

### `UNLOCK_OPTION_SELECTED`

```json
{ "attempt_id": "...", "option_type": "CREDITS_UNLOCK|QUEST_UNLOCK|EMERGENCY_UNLOCK", "option_payload": {} }
```

### `UNLOCK_GRANTED`

```json
{ "grant_id": "...", "app_id": "...", "method": "QUEST", "starts_ts_utc_ms": 1738460000000, "ends_ts_utc_ms": 1738460300000, "attempt_id": "..." }
```

### `UNLOCK_DENIED`

```json
{ "attempt_id": "...", "reason": "INSUFFICIENT_CREDITS|DAILY_LIMIT_REACHED|COOLDOWN_ACTIVE|QR_KEY_NOT_SET|DISALLOWED_IN_HARD_MODE|SAME_APP_CONSECUTIVE_BLOCKED" }
```

### Quest events

`QUEST_STARTED`

```json
{ "quest_session_id": "...", "attempt_id": "...", "quest_type": "COPY_TEXT" }
```

`QUEST_COMPLETED`

```json
{ "quest_session_id": "...", "attempt_id": "...", "quest_type": "COPY_TEXT" }
```

`QUEST_FAILED`

```json
{ "quest_session_id": "...", "attempt_id": "...", "quest_type": "COPY_TEXT", "reason": "TIMEOUT|MISMATCH|USER_CANCELLED|EXPIRED" }
```

`QUEST_CANCELLED`

```json
{ "quest_session_id": "...", "attempt_id": "...", "quest_type": "BREATHING" }
```

`QUEST_EXPIRED`

```json
{ "quest_session_id": "...", "attempt_id": "...", "quest_type": "QR_SCAN" }
```

### Emergency

`EMERGENCY_USED`

```json
{ "attempt_id": "...", "app_id": "...", "delay_seconds": 60, "unlock_minutes": 5 }
```

### Focus lifecycle (recommended)

`FOCUS_STARTED`

```json
{ "session_id": "...", "planned_minutes": 50 }
```

`FOCUS_ENDED`

```json
{ "session_id": "...", "completed": true, "ended_early": false, "actual_minutes": 50 }
```

---

## A8) Data retention & compaction (v1.1)

To prevent unbounded growth:

* Keep `event_log` for **30 days**
* Keep `unlock_attempt`, `unlock_grant`, `quest_session`, `unlock_grant_extension` for **30 days**
* Keep `day_summary` indefinitely
* Keep `focus_session` and `habit_completion` indefinitely (small, user-facing)

A cleanup job runs:

* on app start (if last cleanup > 24h)
* and optionally daily via background task (platform permitting)

---

# Part B — iOS App Group Shared KV contract (non-SQL, required)

This is required for the iOS “real blocker” UX:

**Storage:** App Group Shared UserDefaults (or equivalent shared key-value store)

Keys:

* `ios_last_blocked_app_id` (string)
  Example: `iosToken:AbCDeF...`

* `ios_last_blocked_ts_utc_ms` (string integer)

* `ios_last_blocked_mode_id` (string, optional)

Rules:

* Written by iOS shield/extension when a shield is presented.
* Read by app on launch/resume:

    * if present and recent (e.g., within 10 minutes), route to Unlock Gate for that app_id.

---

# Part C — What was removed from v1.1 (intentionally)

The following v1.0 tables/spec sections are **not** included in v1.1:

* `token_state`, `token_ledger`
* `partner`, `partner_request`
* Firestore cloud schema for partner/tokens

These are planned for later versions once iOS launch is stable.

---
