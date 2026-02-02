## Why these extra docs exist

Your truth stack (Rules Engine Spec > Data Model Spec > Backlog) is excellent for correctness, but **blocker apps have two extra failure modes** that aren’t purely “logic correctness”:

1. **Platform constraints** (iOS Screen Time is powerful but heavily constrained; some UX patterns are not possible).
2. **Store positioning + review clarity** (App Review needs a clean explanation + testing steps).

So these docs exist to prevent agent drift *and* to keep App Review / user UX aligned with what’s actually possible.

## Recommended v1.1 doc set (minimal but complete)

**Truth docs (already in your pack):**

* `Rules Engine Spec v1.1` (behavior truth)
* `Data Model Spec v1.1` (storage truth)
* `Backlog v1.1` (work truth)
* `Agent Prompt Pack v1.1` (execution prompts)

**Context + product docs (add these):**

1. `01_PRODUCT_UX_AND_FLOWS.md`
2. `02_IOS_ENFORCEMENT_SPEC.md`
3. `03_MONETIZATION_AND_GATING.md`
4. `04_APP_REVIEW_AND_PRIVACY_PACKET.md`
5. `05_QA_RELEASE_CHECKLIST.md`

## If you want fewer docs



