# `docs/v1.1/04_APP_REVIEW_AND_PRIVACY_PACKET.md`

## App Review “what this app does” (short)

Earned Access helps users reduce distractions by using iOS Screen Time APIs to block selected apps during user-defined schedules. Users can temporarily unlock apps using earned credits or short interventions.

## What APIs are used (plain English)

* Screen Time app selection (user chooses apps)
* Shielding (system blocks the chosen apps during schedules)

## Important limitation (don’t misrepresent)

* The Screen Time shield cannot open the main app via a button (supported API limitation). ([Apple Developer][1])
  So users unlock by opening the app manually.

## Reviewer test steps (copy/paste)

1. Launch app → complete onboarding
2. Select at least one app to block
3. Create a Mode scheduled for the current time
4. Open blocked app → verify shield appears
5. Return to Earned Access → create a 5‑minute unlock for that app
6. Open app again → verify it opens during unlock
7. Wait for unlock to expire → verify it blocks again

## Privacy posture (v1.1)

* Local-first storage by default
* No cloud sync required for core loop
* Diagnostics export is user-initiated
* No “usage tracking” claims unless you truly implement it

## Claims to avoid in store copy

* “Unbreakable”
* Medical/treatment claims
* “Automatically redirects you out of apps” (not true without Shortcuts)

---

# `docs/v1.1/05_QA_RELEASE_CHECKLIST.md`

## CI gates (every PR)

* Typecheck
* Unit tests (rules truth-table scenarios)
* Lint
* Build verification (dev client)

## iOS enforcement manual checks (pre-release)

* FamilyControls entitlement present for app + extensions
* Picker works, selection persists across reboot
* Shield appears during active schedule
* Unlock grant:

    * allows app
    * expires reliably and re-blocks
* Airplane mode:

    * credits/quests/emergency still work
    * no crashes

## Subscription checks

* Trial starts correctly
* Tier cache updates correctly
* If offline: tier stays last-known-good, with safe fallbacks

## Launch toggles

* Keep enforcement flags explicit:

    * `ios_screentime_enabled = ON` (v1.1 iOS launch)
    * `android_enforcement_enabled = OFF` (until ready)

---

## Final notes on “is this useful to us?”

Yes — these docs eliminate the most dangerous drift vectors:

* **iOS Screen Time UX constraints** (no redirect; don’t build fantasies) ([Apple Developer][1])
* **Grant expiry enforcement** (needs extension scheduling; not just app timers) ([Apple Developer][5])
* **Monetization that doesn’t undermine “real blocker” credibility** (Free baseline still blocks)

---

If you want, I can now:

* convert these into a **single consolidated “Product Spec v1.1”** (to reduce doc count further), *or*
* keep them separate and generate the corresponding `/marketing/` store copy that matches the “no redirect” reality and your tier gates.

[1]: https://developer.apple.com/forums/thread/719905 "Open parent app from ShieldAction … | Apple Developer Forums"
[2]: https://riedel.wtf/state-of-the-screen-time-api-2024/ "Apple’s Screen Time API has some major issues | riedel.wtf"
[3]: https://stackoverflow.com/questions/74429234/open-parent-app-from-shieldaction-extension-in-ios "swift - Open parent app from ShieldAction extension in iOS - Stack Overflow"
[4]: https://developer.apple.com/documentation/screentimeapidocumentation?utm_source=chatgpt.com "Screen Time Technology Frameworks"
[5]: https://developer.apple.com/documentation/deviceactivity/deviceactivityschedule/init%28intervalstart%3Aintervalend%3Arepeats%3Awarningtime%3A%29?utm_source=chatgpt.com "init(intervalStart:intervalEnd:repeats:warningTime:)"
