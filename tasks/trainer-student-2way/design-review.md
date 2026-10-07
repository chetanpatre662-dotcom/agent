# Design Review — FitTrack AI Trainer ⇄ Student 2-Way System (+ Nutrition Add-Food bug)

Reviewer: design-review subagent (fresh read, no authoring context).
Design under review: `.agents/tasks/trainer-student-2way/design.md` (revision pass responding to a prior CHANGES_REQUESTED gate with 2 HIGH / 5 MEDIUM / 4 NIT).
Verdict: **APPROVED** (0 HIGH, 0 MEDIUM, 4 NIT).

## Scope and method

I read the full design, then verified its concrete code claims against the live repository. The task named a worktree at `.worktrees/trainer-student-2way`; I confirmed it does **not** exist on disk (`Test-Path` → False), exactly as the design's "Worktree note" states, so investigation fell back to the live repo (`backend/`, `frontend/`, `firebase/`) that the branch derives from. This is an acceptable, disclosed fallback — not a finding — because the structure is identical and all paths resolve there.

Files read to verify claims:
- `backend/src/services/nutritionService.ts`, `backend/src/utils/nutritionCalc.ts`, `backend/src/validators/nutritionValidators.ts`
- `backend/src/middleware/auth.ts`, `backend/src/services/authService.ts`, `backend/src/repositories/userRepository.ts`, `backend/src/routes/auth.routes.ts`
- `backend/src/config/firebase.ts`, `backend/src/services/progressPhotoService.ts`
- `firebase/firestore.rules`
- `frontend/lib/core/routing/app_router.dart`, `frontend/lib/core/routing/route_names.dart`
- `frontend/lib/features/auth/data/auth_repository.dart`
- `frontend/lib/models/food.dart`, `frontend/lib/features/nutrition/presentation/widgets/add_food_sheet.dart`, `frontend/lib/features/nutrition/providers/nutrition_providers.dart`, `frontend/lib/features/nutrition/data/nutrition_repository.dart`, `frontend/lib/features/nutrition/presentation/nutrition_screen.dart` (grep)
- `frontend/lib/features/shell/home_shell.dart`

## Summary judgment

This is a strong revision pass. Every blocking issue the design claims to resolve in §12 is genuinely resolved, and — crucially — the design's code-level assertions are **accurate against the real source**, not assumed. The two previously-HIGH items (nutrition root cause; router role-branch ordering + role in verify response) are now correct and verifiable. The security model (server-mediated cross-user reads, fail-closed rules, 404-not-403 for unowned students, owner-only Storage + signed URLs) is sound and prevents one trainer from reading another's students and one student from reading another's data. No HIGH or MEDIUM findings remain. The four NITs below are non-blocking.

---

## Verified assumptions (checked against real code)

1. **Nutrition root cause (§7.1) is real and correctly attributed.** `nutritionService.getDay()` maps each Firestore row into a `FoodLogEntry` with exactly `{ id, mealType, name, quantity, calories, protein, carbs, fat, fiber, sugar, sodium }` and **omits `mealId`/`mealName`**. `nutritionCalc.ts` `FoodLogEntry` has no `mealId`/`mealName` fields. Confirmed.
2. **Write side is correct (§7.1).** `add_food_sheet.dart` `_buildEntry()` sets `mealId: widget.mealId` and `mealName: widget.mealName` on both the selected-food and custom-food paths; `food.dart` `FoodLogEntry.toJson()` emits `mealId`/`mealName` when non-null; `foodLogUpsertSchema` accepts both (`.nullable().optional()`). The association is persisted. Confirmed.
3. **Symptom confirmed as the reported bug.** `nutrition_screen.dart` renders built-in meals via `entriesFor(meal)` (keyed on `mealType`, unaffected) and named custom meals via `entriesForCustomMeal(meal)`, which matches `e.mealId == meal.id || (e.mealId == null && e.mealName == meal.name)`. With both fields null on read, a food logged to a **named custom meal** matches no section and disappears — exactly the "added food doesn't appear" report. Confirmed.
4. **No idempotency currently (§7.4).** `food.dart` `toJson()` omits `id`; `nutritionService.addFood` always calls `newLogId(uid)`; `foodLogUpsertSchema` has no `clientId`. The `_saving` guard exists but does not cover slow-network re-taps. Confirmed; the client-supplied-id fix is correct and uses `setLog`'s existing `{ merge: true }`.
5. **Date-key symmetry (§7.3).** `nutritionDateProvider` produces one local key; `AddFoodSheet` sends `widget.dateKey`; `nutritionDayProvider` reads the same provider; `addFood`/`getDay` therefore use the same key. Confirmed non-issue.
6. **Invalidate-before-pop (§7.5).** `add_food_sheet.dart` `_add()` already does `ref.invalidate(nutritionDayProvider)` then `Navigator.pop()`; `nutritionDayProvider` is a non-family `FutureProvider` so the plain invalidate is correct. Confirmed.
7. **Router ordering claim (§6.1) is correct.** `app_router.dart` computes `onboarded = account?.onboardingCompleted ?? false` and sends non-onboarded users to `Routes.onboarding`. A seeded trainer with `onboardingCompleted:false` would be mis-sent to student onboarding if the role branch were placed after the gate; inserting it immediately after the `accountAsync.isLoading`/`hasError` holds and before `final onboarded = ...` is the correct, verifiable insertion point. Confirmed.
8. **Role wiring points (§5.1) are accurate.** `authService.verifyAndSync` returns `userRepository.ensureAccount(...).account` verbatim with no `role`; `AccountInfo` has no `role` field today and defaults are parsed with `?? '...'`. Injecting the resolved role into the returned account and parsing `account['role'] as String? ?? 'student'` fits the existing shapes exactly. Confirmed.
9. **No-write-on-unchanged-reopen invariant (§2.5) exists.** `ensureAccount` only patches `email`/`emailVerified` (with `{ merge: true }`) and returns the existing doc otherwise, never touching `role`. A computed-only `resolveRole` preserves this. Confirmed.
10. **Fail-closed rules (§4) baseline exists.** `firestore.rules` has owner-only `users/{uid}/**` and a default-deny catch-all; reference collections are read-only. The new client-deny for `trainers`/`referralCodes`/`trainerLinks` keeps student data non-public and forces cross-user reads through the authorized backend. Confirmed sound.
11. **Signed-URL credential constraint (§2.3) basis exists.** `config/firebase.ts` initializes with `admin.credential.cert(...)` in both branches (inline key or service-account JSON) — both key-bearing, so `getSignedUrl` V4 works with the current config. `progressPhotoService` already keeps bytes in Storage with the `assertOwnedPath` owner-only invariant, which the signed-URL approach reuses without loosening `storage.rules`. Confirmed.
12. **`/api/auth/link-trainer` works pre-verification (§6.4).** `auth.routes.ts` applies only `authenticate` (token verify, no email-verified check); a freshly-registered, signed-in-but-unverified user has a valid token, so the link call succeeds before the verify-email redirect. Confirmed.
13. **Student shell untouched + Midnight theme (§6.2).** `home_shell.dart` is a self-contained 5-tab shell with its own custom Midnight-styled nav bar using `AppColors`/`AppShadows`. A separate trainer shell leaves it unmodified. Confirmed.

## Unverified / wrong assumptions

None material. The design's claims that I could check all held. Items I could not fully verify are future-build details that cannot exist yet (the new trainer slice, the seed script, the signed-URL method), which is expected for a design-only step; the design specifies them concretely enough to implement.

---

## Findings

### NIT 1 — `firestore.rules` deny blocks are redundant with the existing default-deny
`firestore.rules` already ends with `match /{document=**} { allow read, write: if false; }`, so the three new explicit `allow read, write: if false;` matches for `trainers`/`referralCodes`/`trainerLinks` (§4) are redundant. They are not wrong — explicitness documents intent and guards against a future top-level `allow` creeping above the catch-all — but the design should state they are deliberately explicit-for-documentation rather than functionally required, so the implementer doesn't believe the collections would be client-readable without them.
Fix: add a one-line note in §4: "These three matches are redundant with the trailing default-deny and are kept only as explicit, self-documenting guards; removing them changes nothing functionally."

### NIT 2 — Signed-URL "ADC-without-key" caution does not match this repo's config
§2.3 warns that under Application Default Credentials without a private key `getSignedUrl` throws, and forbids CI tests that assert real signed URLs. That caution is reasonable, but `config/firebase.ts` has **no** ADC path: both credential branches call `admin.credential.cert(...)` (inline key or key-bearing JSON). So in this codebase the only way to hit keyless creds is to change the config. The guidance is harmless but slightly over-stated.
Fix: reword §2.3 to "the current `firebase.ts` always uses key-bearing cert credentials, so signing works in all configured environments; keep the per-photo `url:null` fallback and the no-real-signed-URL-in-emulator rule only as defense for a future ADC/emulator setup."

### NIT 3 — "Trainer email if shared" references a sharing flag that is not modeled
§5.3 `GET /api/student/trainer` returns the trainer's "email if shared," but §3.3 (`trainers/{trainerId}`) defines a plain `email` field with no visibility/sharing flag, and the only sharing toggle in the design is `shareProgressWithTrainer` (photos, student→trainer direction). The "if shared" qualifier is therefore ambiguous — there is no mechanism to drive it.
Fix: pick one and state it. Either (a) drop "if shared" and always return the trainer's contact email (trainer contact info is not private student data), or (b) add an explicit `trainers/{trainerId}.contactEmailPublic: boolean` (default true) and gate on it. (a) is simpler and matches requirement 7's "Trainer email/contact information if currently supported."

### NIT 4 — Register→link race and SnackBar teardown is handled but the ordering guarantee should be pinned
§6.4 correctly routes the invalid-code message to the verify-email screen via `linkTrainerResultProvider` because the register screen is torn down by the auth-state redirect. The remaining subtlety: `linkTrainer(code)` is awaited in `_submit` after `register()`, but the router's redirect fires off the auth-state change independently, so the register screen may be disposed while the link call is still in flight. The design's provider-write approach survives that (the awaited call still completes and writes the provider even if the widget is gone), but only if `linkTrainer` writes the provider through a `Ref`/notifier that outlives the screen (not through the screen's own `ref` after dispose).
Fix: in §6.4 state explicitly that `linkTrainer` writes `linkTrainerResultProvider` via a provider/notifier (e.g. `ref.read(container)` captured before navigation or a `Notifier`), not via a `WidgetRef` that may be disposed, so a mid-flight teardown cannot drop the result.

---

## Verdict rationale

HIGH = 0, MEDIUM = 0 → **APPROVED**. The four NITs are documentation/robustness clarifications that do not block implementation. The previously-blocking HIGH items (nutrition root cause misattribution; router branch contradiction + role missing from verify response) and the five MEDIUM items are all genuinely resolved and verified against real code.
