# Implementation Plan (v1.2)

## Source Of Truth
- Primary: `docs/1.2.md` (consolidated v1.2 spec).
- If conflicts remain: `Rules Engine Spec v1_1` > `Data Model Spec v1_1` > `Backlog v1_1`.
- iOS enforcement constraints from `02_IOS_ENFORCEMENT_SPEC.md` and `04_APP_REVIEW_AND_PRIVACY_PACKET.md` must be honored.

## How To Use This Plan
- Read this file first after any context reset.
- Follow phases in order.
- Update the **Progress Tracker** after each PR merge (or when a phase starts/finishes).
- Keep PRs **mid-sized** and cohesive (target ~300–700 LOC diff; avoid tiny “one-screen” PRs).

## Progress Tracker (update after each PR)
- Current phase: Phase 1 (pending)
- Current branch: emdash/implement-7ca
- Current PR: 
- Last completed step: Phase 0 — Pre-flight alignment (decisions locked, docs updated)
- Next step: Phase 1 — Foundation + SQLite layer (E0 + E1)
- Tests run: None (docs-only updates)

### Phase Status
- Phase 0: COMPLETED
- Phase 1: NOT_STARTED
- Phase 2: NOT_STARTED
- Phase 3: NOT_STARTED
- Phase 4: NOT_STARTED
- Phase 5: NOT_STARTED
- Phase 6: NOT_STARTED
- Phase 7: NOT_STARTED

### PR Log (append newest at top)
- (empty)

---

## Phase 0 — Pre-flight Alignment (no code changes)
Goal: confirm scope, version source, and repo expectations before implementation.
- Confirm v1.2 is the active source of truth.
- Monetization (RevenueCat / tiers) is in scope for initial release.
- Confirm PR sizing expectations and review cadence.

Deliverable:
- Updated Progress Tracker fields.

---

## Phase 1 — Foundation + SQLite Layer (E0 + E1)
Goal: app boots reliably with correct DB location + schema + repositories.
- Project structure aligned with spec.
- Feature flags and boot sequence (DB init, ensure singletons, ensureDayState).
- SQLite schema and migrations per Data Model Spec v1.1.
- Repos for singletons, modes, focus, habits, unlocks, quests, day_summary, event_log.
- `computeDayId()` + unit tests.
- DB location uses iOS App Group container path.

Definition of Done:
- App boots to shell screens.
- All singleton rows ensured.
- Migration v1 applied with `schema_version = 1`.
- Tests pass for `computeDayId` and key DB invariants.

---

## Phase 2 — Rules Engine Core + Truth-Table Tests (E2)
Goal: deterministic rules engine and atomic mutations per spec.
- Day rollover + streak qualification + bonus.
- Credits earn/spend, anti-gaming for habits.
- Mode activation + precedence.
- `evaluateAppAccess()` and unlock option snapshots.
- Unlock attempt lifecycle; grant creation/extension.
- Quest + emergency rules.
- Full truth-table tests from Rules Engine Spec.

Definition of Done:
- All required tests pass.
- Event log emitted for every mutation type.
- No UI dependency in engine code.

---

## Phase 3 — Core UI Shell + Onboarding + Modes (E3 + E4)
Goal: user can onboard and create real blocking setup.
- Navigation tabs (Modes, Focus, Unlocks, Progress, Settings).
- iOS-first onboarding flow.
- Screen Time permission explanation and flow (native hooks stubbed if needed).
- Global “Distracting Apps” management and mode CRUD.
- Mode schedule editor + precedence explanations.
- App selection via iOS picker (token-based).

Definition of Done:
- New user can create first mode and select apps.
- UX matches “manual open app to unlock” reality.

---

## Phase 4 — Focus + Habits + Unlock Gate UX (E5 + E6 + E7)
Goal: core loop UX fully operational with engine integration.
- Focus timer + history.
- Habits CRUD + completion + suspension UX.
- Unlock Gate for credits, quests, emergency.
- Active grants list + countdown.
- Unlock attempt history.

Definition of Done:
- Unlock Gate creates attempts and grants via engine.
- All edge cases show disabled reasons.

---

## Phase 5 — iOS Enforcement (E8)
Goal: real OS-level blocking on iOS using Screen Time APIs.
- Entitlements + App Group config.
- Authorization flow.
- FamilyControls picker integration and token encoding/decoding.
- DeviceActivity monitoring for schedules and grant expiry.
- Shield UI + action delegate writes last blocked app to App Group.
- App auto-navigates to Unlock Gate for last blocked app.

Definition of Done:
- Manual test checklist passes (shield, unlock, re-block).
- At least one automated test for token encoding or shield set computation.

---

## Phase 6 — Progress, Settings, QA/Privacy (E10 + E11 + docs)
Goal: app is store-review ready and stable.
- Progress dashboard + weekly review.
- Settings screen (quest toggles, Screen Time status, emergency read-only constants).
- Diagnostics export + retention cleanup.
- Privacy and terms screens.
- QA release checklist doc updated for current build.

Definition of Done:
- Progress + settings features are functional offline.
- Retention job runs safely.

---

## Phase 7 — Monetization (In Scope) (E9)
Goal: tier gating + paywall if required for launch.
- RevenueCat integration (or local entitlements only).
- Tier gating enforced in engine + UI.
- Trial flow if enabled.
- Deterministic downgrade behavior on trial end/lapse (mode selection, 3-app cap, unlock duration gating, honor active grants).
- Pro purchase options: monthly + annual (trial) + lifetime (non-consumable).
- Trial/paywall triggers: 4th distracting app, 2nd mode, 15/30 unlocks, Strict/Hard enable.

Definition of Done:
- Free vs Pro features correctly gated.
- Offline behavior uses last-known entitlements safely.

---

## PR Strategy
- Each phase maps to 1–2 mid-sized PRs.
- Every PR must update the Progress Tracker.
- Every PR must include:
  - Summary of changes.
  - Tests run (or explicit note if not run).
  - Any follow-ups required.

## Required Post-Reset Steps (for any agent)
1. Read `docs/IMPLEMENTATION_PLAN.md`.
2. Check Progress Tracker for current phase/next step.
3. Read relevant spec docs before coding.
4. Continue with the “Next step” in tracker.
