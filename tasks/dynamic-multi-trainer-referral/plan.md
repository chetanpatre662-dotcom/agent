# Implementation Plan — Dynamic Multi-Trainer Referral System

Source of truth: `c:\Users\cheta\OneDrive\Desktop\fit\.agents\tasks\dynamic-multi-trainer-referral\design.md` (APPROVED). Hardcoded-touchpoint inventory: `c:\Users\cheta\OneDrive\Desktop\fit\.agents\tasks\dynamic-multi-trainer-referral\inventory.md`. Prior architecture: `.agents\tasks\trainer-student-2way\{design.md,plan.md}`.

Work IN PLACE on branch `feat/trainer-student-2way` in both repos. Do NOT create worktrees/branches, do NOT force-push, do NOT touch git config. Commit scoped files per repo (never `git add -A`); leave the large untracked frontend baseline alone. `firebase/` is unversioned — flag any change for user review, do not commit it.

Verification gate — backend (`c:\Users\cheta\OneDrive\Desktop\fit\backend`): `npm run typecheck`, `npm run lint`, `npm test`. Frontend (`c:\Users\cheta\OneDrive\Desktop\fit\frontend`): `flutter analyze`, `flutter test`. All commands run from the respective repo root.

Key design facts the implementer must honor:
- `trainerLinks/{studentUid}.trainerId` is the authoritative relationship key — NEVER keyed by code or email. Changing/regenerating a code must never read or write `trainerLinks`.
- `trainers/{trainerId}.status === 'active'` is the single source of truth for trainer-active; `active` is a derived mirror written as `status === 'active'`, never read for a decision.
- `referralCodes/{codeLower}` is a lookup index ONLY; the trainer record is authoritative for ownership.
- Shared claimability predicate: a code is claimable by trainer T iff the doc does not exist OR is owned by T; unavailable iff it exists with a different trainerId regardless of `active`.
- The stored `totalStudents` counter is advisory; `getProfile` overwrites it with the live `countActiveStudents`. The switch path writes NO counter.
- `dreamphysics` is grandfathered by migration (no validator run on migration) and reserved so no OTHER trainer can reclaim it.

---

## A. Backend — remove hardcoded dreamphysics and build the referral-code engine

- [ ] 1. Remove the fixed-trainer seed and its npm script; update the stale validator comment.
      Repurpose `seedTrainer.ts` into a generic no-arg-safe dev bootstrap: drop the `REFERRAL_CODE='dreamphysics'` / `TRAINER_NAME='Dream Physics'` constants and the privileged `SEED_TRAINER_UID`-as-sole-mechanism; promote a given/created UID to trainer and assign a GENERATED code via the production generator (step 3). Update the `src/validators/trainerValidators.ts` doc comment that references "the only seeded code today is 'dreamphysics'".
      Files: `c:\Users\cheta\OneDrive\Desktop\fit\backend\src\scripts\seedTrainer.ts`, `c:\Users\cheta\OneDrive\Desktop\fit\backend\src\validators\trainerValidators.ts`, `c:\Users\cheta\OneDrive\Desktop\fit\backend\package.json` (keep `seed:trainer` only as the generic helper).
      Verify: `cd c:\Users\cheta\OneDrive\Desktop\fit\backend && npm run typecheck && npm run lint` pass; `grep -rn "dreamphysics" src --include=*.ts` returns matches ONLY in the reserved-word list (step 6) and migration/test fixtures — never an `if (code === 'dreamphysics')` branch and never a hardcoded seed constant.

- [ ] 2. Create the pure referral-code generator util.
      `generateCandidate(rng?)`: fixed prefix from a small set (`FIT`/`TRN`/`GYM`/`COACH`) + random suffix from an unambiguous alphabet (uppercase letters + digits, excluding `0 O 1 I`), length 7–8, drawing via Node `crypto.randomInt` with an injectable RNG for deterministic tests. `normalize(code)`: `trim().toLowerCase()`. No uniqueness logic here (that lives in the repo transaction).
      Files: `c:\Users\cheta\OneDrive\Desktop\fit\backend\src\utils\referralCode.ts`.
      Verify: unit test in step 10 asserts non-sequential output, alphabet exclusion, length bounds, determinism with injected RNG — `npm test` green.

- [ ] 3. Extend `trainerRepository` with the referral-code index operations (supersede `setReferralCode`).
      Add `getByReferralCodeDoc(codeLower): { trainerId, active } | null`; `claimReferralCode(codeLower, displayCode, trainerId, tx)` enforcing the shared claimability predicate and writing `{ trainerId, code, active:true, createdAt (first write only), updatedAt }`; `deactivateReferralCode(codeLower, tx)` → `{ active:false, updatedAt }`; `setActiveReferralCodeOnTrainer(trainerId, displayCode, codeLower, tx)` → update `trainers/{trainerId}` `referralCode`/`referralCodeLower`/`referralCodeUpdatedAt`. Remove `setReferralCode` or reduce it to a thin wrapper over `claimReferralCode` (it is unused by new paths). All reads inside callers' transactions precede writes per design §2.5.
      Files: `c:\Users\cheta\OneDrive\Desktop\fit\backend\src\repositories\trainerRepository.ts`.
      Verify: `npm run typecheck` passes; repo-level behavior asserted via service tests in step 10.

- [ ] 4. Add the referral-code validators.
      `referralCodeSchema` (`code` required, `trim()`, length 4–20, regex `^[A-Za-z0-9]+$`, reserved-word rejection case-insensitive against `admin`,`trainer`,`fittrack`,`support`,`null`,`undefined`,`dreamphysics`); `availabilityQuerySchema` (`code`, same format rules); `registerTrainerSchema` (`name` 1–80, no referral-code field); extend `linkTrainerSchema` with optional `confirmSwitch: z.boolean().optional()`.
      Files: `c:\Users\cheta\OneDrive\Desktop\fit\backend\src\validators\trainerValidators.ts`.
      Verify: validator unit tests in step 10 cover format/reserved/confirmSwitch — `npm test` green.

- [ ] 5. Extend `trainerService` with the referral-code lifecycle methods.
      Add `ensureTrainerAccount(uid, {name,email,photoUrl})` (idempotent: upsert `users/{uid}.role='trainer'` + `trainers/{uid}` with `userId`, `active:true`, `status:'active'`, `createdAt` first-write; generate+claim a unique code only if none exists — no duplicate trainer/code on re-run); `getReferralCode(trainerId)`; `generateReferralCode(trainerId)` (outer candidate loop ≤5, each attempt one all-reads-before-writes transaction that claims new + deactivates old + updates trainer atomically; `503` only if all attempts collide); `setCustomReferralCode(trainerId, desiredCode)` (validate, then the inner transaction once; collision → `409 ConflictError`); `checkAvailability(desiredCode)` (validate format/reserved, then shared predicate, read-only). Reuse `ConflictError`/`ServiceUnavailableError`/`BadRequestError` from `utils/errors.ts`.
      Files: `c:\Users\cheta\OneDrive\Desktop\fit\backend\src\services\trainerService.ts`.
      Verify: service unit tests in step 10 — `npm test` green.

- [ ] 6. Add the one-active-trainer switch guard to `linkStudent`.
      Signature → `linkStudent(studentUid, referralCode, opts?: { confirmSwitch?: boolean })`. Resolve code via `getByReferralCodeDoc` (missing OR `active:false` → `{linked:false, reason:'invalid_code'}`); verify the resolved account is a trainer (`roleService.resolveRole === 'trainer'` and `trainers/{id}` exists and `status==='active'`), else `invalid_code`; trainer-role caller → `409` (unchanged). On `trainerLinks/{studentUid}`: no active link → existing inline `tx.set` create path (+1 counter, unchanged); same trainer → idempotent `{linked:true, alreadyLinked:true}` no double count; different trainer and `confirmSwitch!==true` → `{linked:false, reason:'already_linked', currentTrainer:{trainerId,name|null}, requestedTrainer:{trainerId,name}}` with NO write; different trainer and `confirmSwitch===true` → transaction updating ONLY the link row (`trainerId`→B, `updatedAt`) and `profile.trainerId`, NO counter mutation. Extend `LinkResult` union with the new reasons/fields.
      Files: `c:\Users\cheta\OneDrive\Desktop\fit\backend\src\services\trainerService.ts`.
      Verify: switch-guard unit tests in step 10 — `npm test` green.

- [ ] 7. Add trainer referral-code endpoints on the `/api/trainer` slice.
      Controller handlers + routes behind `authenticate`+`requireTrainer`: `GET /api/trainer/referral-code` → `getReferralCode`; `POST /api/trainer/referral-code` → `generateReferralCode`; `PATCH /api/trainer/referral-code` (body `referralCodeSchema`) → `setCustomReferralCode`; `GET /api/trainer/referral-code/availability?code=` (query `availabilityQuerySchema`) → `checkAvailability`. Use `ok()` envelope.
      Files: `c:\Users\cheta\OneDrive\Desktop\fit\backend\src\controllers\trainerController.ts`, `c:\Users\cheta\OneDrive\Desktop\fit\backend\src\routes\trainer.routes.ts`.
      Verify: route-matrix test in step 11 asserts 401/403(student)/200(trainer) — `npm test` green.

- [ ] 8. Add `POST /api/auth/register-trainer` and evolve `link-trainer`.
      `register-trainer` (authenticated, body `registerTrainerSchema`) → `ensureTrainerAccount` using the verified token's uid/email/name; role set server-side only. Evolve `authController.linkTrainer` to pass `confirmSwitch` from the body through `linkStudent`.
      Files: `c:\Users\cheta\OneDrive\Desktop\fit\backend\src\controllers\authController.ts`, `c:\Users\cheta\OneDrive\Desktop\fit\backend\src\routes\auth.routes.ts`.
      Verify: integration test in step 11 — `register-trainer` 401 no token / 200 promotes / idempotent re-call; `npm test` green.

- [ ] 9. Add `POST /api/student/connect-trainer` (thin alias) on the `/api/student` slice.
      Delegates to `linkStudent` with the same body/semantics as `link-trainer` (reuse `linkTrainerSchema` incl. `confirmSwitch`). Keep `GET /api/student/trainer` unchanged.
      Files: `c:\Users\cheta\OneDrive\Desktop\fit\backend\src\controllers\studentController.ts`, `c:\Users\cheta\OneDrive\Desktop\fit\backend\src\routes\student.routes.ts`.
      Verify: integration test in step 11 — connect valid/invalid/already-linked/confirmed-switch; `npm test` green.

- [ ] 10. Add/extend backend unit tests (in-memory Firestore mock like `trainerService.test.ts`).
      Cover: generator (non-sequential, alphabet, length, deterministic RNG); `claimReferralCode` (first claim ok, other-trainer claim fails, same-trainer re-claim no-op); `generateReferralCode`/`setCustomReferralCode` (new active, old deactivated, trainer updated, `trainerLinks` untouched before/after); forced single collision retries to a fresh candidate and leaves old code `active` unchanged on the aborted attempt; `checkAvailability` (available/taken-by-other/reserved/invalid); `linkStudent` switch guard (no link→links; same trainer→no double count; different no-confirm→`already_linked` + NO write; different with confirm→link `trainerId`→B + profile re-mirrored + neither stored `totalStudents` written + live count A−1/B+1; `already_linked` name null when A missing); `ensureTrainerAccount` idempotency (no duplicate trainer/code on re-run); validators (`referralCodeSchema` format/reserved, `linkTrainerSchema` with `confirmSwitch`). Update existing `trainerService.test.ts` fixtures to arbitrary trainer names/codes rather than the single `dreamphysics`.
      Files: `c:\Users\cheta\OneDrive\Desktop\fit\backend\src\__tests__\trainerService.test.ts`, new `c:\Users\cheta\OneDrive\Desktop\fit\backend\src\__tests__\referralCode.test.ts`.
      Verify: `npm test` — all green.

- [ ] 11. Add/extend backend route-integration tests.
      Extend `trainerRoutes.test.ts` for the four referral-code endpoints (401/403-student/200-trainer; generate changes the code; availability returns `{available,reason}`); add coverage for `register-trainer` (401/200/idempotent) and `connect-trainer` (valid/invalid/already-linked/confirmed-switch). Update the `dreamphysics` fixtures to arbitrary codes. Keep the full existing `/api/trainer/*` authorization matrix green.
      Files: `c:\Users\cheta\OneDrive\Desktop\fit\backend\src\__tests__\trainerRoutes.test.ts`.
      Verify: `npm test` — all green.

- [ ] 12. Add the idempotent migration script.
      `migrateReferralCodes.ts`: Admin SDK, `hasFirebaseCredentials()` guard; backfill `trainers/*` (`referralCode`,`referralCodeLower`,`active:(status==='active')`,`referralCodeUpdatedAt`,`createdAt`,`userId`) and `referralCodes/*` (`code`,`active:true`,`createdAt`,`updatedAt`, preserve `trainerId`) WITHOUT running the format/reserved validator (grandfather `dreamphysics`); never touch `trainerLinks`. Add npm script `migrate:referral-codes`. This is run manually, not in CI.
      Files: `c:\Users\cheta\OneDrive\Desktop\fit\backend\src\scripts\migrateReferralCodes.ts`, `c:\Users\cheta\OneDrive\Desktop\fit\backend\package.json`.
      Verify: `npm run typecheck && npm run lint` pass (script compiles; no credential-dependent run in CI).

- [ ] 13. Confirm the nutrition Add-Food fix did not regress and commit the backend changes.
      Run the backend verification gate in full. Confirm `nutritionService.test.ts` (getDay preserves `mealId`/`mealName`; `clientId` idempotency; `clientId` never persisted) is green and untouched.
      Files: commit scoped backend files only (routes, controllers, services, repositories, validators, utils, scripts, __tests__, package.json) — never `git add -A`.
      Verify: `cd c:\Users\cheta\OneDrive\Desktop\fit\backend && npm run typecheck && npm run lint && npm test` all pass; `git status` shows only intended files staged.

## B. Frontend — account-type selection, trainer code management, connect-from-profile

- [ ] 14. Extend the auth + trainer data layers with the new endpoints and models.
      `auth_repository.dart`: add `registerTrainer(name)` → `POST /api/auth/register-trainer`; add optional `confirmSwitch` to `linkTrainer` (default false). `trainer_repository.dart`: add `getReferralCode()`, `generateReferralCode()`, `setCustomReferralCode(code)`, `checkAvailability(code)`, `connectTrainer(code,{confirmSwitch})`, each returning `Result<T>` and parsing the `{success,data}` envelope. New models: `ReferralCodeInfo{code,active,updatedAt}`, `AvailabilityResult{available,reason}`, `ConnectResult` (superset of `LinkTrainerResult` carrying `reason:'already_linked'` + `currentTrainer`/`requestedTrainer`).
      Files: `c:\Users\cheta\OneDrive\Desktop\fit\frontend\lib\features\auth\data\auth_repository.dart`, `c:\Users\cheta\OneDrive\Desktop\fit\frontend\lib\features\trainer\data\trainer_repository.dart`, `c:\Users\cheta\OneDrive\Desktop\fit\frontend\lib\features\trainer\data\trainer_models.dart`.
      Verify: `cd c:\Users\cheta\OneDrive\Desktop\fit\frontend && flutter analyze` passes.

- [ ] 15. Add the account-type selector (USER vs TRAINER) to the register screen + a RegisterTrainerController.
      Add a local `_AccountType` state with a segmented/two-card selector above the form fields built from existing `AppColors`/`AppSpacing` (selected state visually obvious, Midnight Energy look). USER (default): unchanged form incl. optional referral field and existing `linkTrainerControllerProvider` flow. TRAINER: hide the referral field; after successful `register()`, fire `RegisterTrainerController.register(name)` (a root-container controller analogous to `LinkTrainerController`) hitting `registerTrainer`. Role becomes trainer server-side; router routes to the trainer shell on next verify — no router change.
      Files: `c:\Users\cheta\OneDrive\Desktop\fit\frontend\lib\features\auth\presentation\register_screen.dart`, `c:\Users\cheta\OneDrive\Desktop\fit\frontend\lib\features\auth\providers\auth_controller.dart`.
      Verify: `flutter analyze` passes; widget test in step 18 asserts field visibility toggles by type.

- [ ] 16. Add trainer referral-code management to the Profile `_ReferralCard`.
      Keep Copy. Add a "Change / Generate New" action opening a bottom sheet: "Generate a new code" (`generateReferralCode`) and "Choose a custom code" (field + "Check Availability" → `checkAvailability` with live ✓ Available / ✗ Taken / format error, then "Save" → `setCustomReferralCode`). On success invalidate `trainerProfileProvider` and show a SnackBar. Card still reads the code dynamically from `TrainerProfile.referralCode`. Show a live Total Students value (already from `profile.totalStudents` via the live backend count) — ensure it is surfaced on the card/section.
      Files: `c:\Users\cheta\OneDrive\Desktop\fit\frontend\lib\features\trainer\presentation\trainer_profile_screen.dart`, (if a provider is needed) `c:\Users\cheta\OneDrive\Desktop\fit\frontend\lib\features\trainer\providers\trainer_providers.dart`.
      Verify: `flutter analyze` passes; widget test in step 18 drives generate/custom flow via overridden providers.

- [ ] 17. Add connect-after-registration + switch confirmation to the student Profile "My Trainer" section.
      In the `MyTrainer.none` state, replace the static "Enter a code when you register" copy with a "Connect to Trainer" affordance opening a sheet (code field → `connectTrainer`). On `invalid_code` show "Invalid trainer referral code"; on `already_linked` show a switch dialog ("You're connected to {currentTrainer.name}. Switch to {requestedTrainer.name}?" with `[Cancel] [Switch Trainer]`; degrade name to "your current trainer" when null) that re-calls `connectTrainer(confirmSwitch:true)`; on success show "Connected to {trainer}" and invalidate `myTrainerProvider`. Connected state unchanged. Empty code → no call.
      Files: `c:\Users\cheta\OneDrive\Desktop\fit\frontend\lib\features\profile\presentation\profile_screen.dart`.
      Verify: `flutter analyze` passes; widget test in step 18 covers invalid/already-linked/switch outcomes.

- [ ] 18. Add/extend Flutter widget tests and update dreamphysics fixtures.
      New/updated tests: account-type selector toggles referral-field visibility (USER shows, TRAINER hides); trainer-profile change/generate updates the displayed code via overridden providers; student connect-from-profile shows invalid/already-linked/switch outcomes; register-as-trainer routes to the trainer shell (role-driven, extend `trainer_routing_test.dart`). Update `trainer_routing_test.dart`, `trainer_dashboard_test.dart`, `register_referral_test.dart` to use arbitrary trainer names/codes rather than the single `Dream Physics`/`dreamphysics` fixture.
      Files: `c:\Users\cheta\OneDrive\Desktop\fit\frontend\test\register_referral_test.dart`, `c:\Users\cheta\OneDrive\Desktop\fit\frontend\test\trainer_routing_test.dart`, `c:\Users\cheta\OneDrive\Desktop\fit\frontend\test\trainer_dashboard_test.dart`, new `c:\Users\cheta\OneDrive\Desktop\fit\frontend\test\account_type_selector_test.dart`, new `c:\Users\cheta\OneDrive\Desktop\fit\frontend\test\trainer_referral_manage_test.dart`, new `c:\Users\cheta\OneDrive\Desktop\fit\frontend\test\connect_trainer_test.dart`.
      Verify: `cd c:\Users\cheta\OneDrive\Desktop\fit\frontend && flutter test` — all green.

- [ ] 19. Run the frontend verification gate and commit the scoped frontend changes.
      Confirm the Add-Food frontend fix (`add_food_sheet_test.dart`, `food_model_test.dart`) stays green.
      Files: commit scoped frontend lib + test files only — never `git add -A`; leave the large untracked baseline alone.
      Verify: `cd c:\Users\cheta\OneDrive\Desktop\fit\frontend && flutter analyze && flutter test` pass; `git status` shows only intended files staged.

## C. Firebase config + final integration

- [ ] 20. Confirm firestore rules + indexes need no change (and flag any edit for user review).
      Per design §3.5/§4.8 no rule or index change is required: `trainers`/`referralCodes`/`trainerLinks` stay fail-closed (`allow read, write: if false`), and the existing `trainerLinks (trainerId ASC, status ASC, updatedAt DESC)` composite index covers the only cross-trainer query. Verify the files already satisfy this; if any edit is genuinely needed, make it in place in `c:\Users\cheta\OneDrive\Desktop\fit\firebase\` and flag it for user review/deploy (do NOT commit — the dir is unversioned).
      Files: `c:\Users\cheta\OneDrive\Desktop\fit\firebase\firestore.rules`, `c:\Users\cheta\OneDrive\Desktop\fit\firebase\firestore.indexes.json`.
      Verify: read both files and confirm the dynamic model is fully covered; state in the report that no change was required (or list exactly what changed and that it was flagged).

- [ ] 21. Final cross-repo verification.
      Run both verification gates once more to confirm the integrated system builds and passes end-to-end.
      Verify: `cd c:\Users\cheta\OneDrive\Desktop\fit\backend && npm run typecheck && npm run lint && npm test` all pass AND `cd c:\Users\cheta\OneDrive\Desktop\fit\frontend && flutter analyze && flutter test` all pass.

---

## Assumptions / notes
- `seed:trainer` is repurposed (not deleted) as a generic dev helper; either choice satisfies the requirement.
- Trainer profile photo at registration is deferred (`photoUrl:null` until set); design §8.
- `firebase/` is unversioned — no rule/index change is required by the design; any edit is flagged for the user, never committed.
- Branches are left for the user to push/merge.
