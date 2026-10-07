# Implementation Plan — FitTrack AI Trainer ⇄ Student 2-Way System (+ Nutrition Add-Food bug)

Source of truth: the approved design at `.agents/tasks/trainer-student-2way/design.md` (review APPROVED in `design-review.md`). This plan sequences that design into ordered, independently verifiable steps.

## Environment & conventions (read before starting)

- **No worktree exists.** Per the `setup` step, work in-place on the live repo: backend at `c:\Users\cheta\OneDrive\Desktop\fit\backend`, Flutter at `c:\Users\cheta\OneDrive\Desktop\fit\frontend`, Firebase config at `c:\Users\cheta\OneDrive\Desktop\fit\firebase`. Both repos are on branch `feat/trainer-student-2way`. `firebase/firestore.rules` is edited directly (unversioned; flag for user review). The `.worktrees/trainer-student-2way` path from the task prompt does NOT exist — ignore it.
- **Backend** (`backend/`): Node 20 + Express + TypeScript (ESM, `.js` import specifiers) + Firebase Admin + Zod + pino. Layering `routes → controllers → services → repositories`, Zod `validators`, shared `middleware/auth.ts` (`authenticate` verifies the Firebase ID token, sets `req.uid`; `requireUid(req)` returns it). Response envelope via `ok(res, data)` → `{ success, data }`; errors via `utils/errors.ts` subclasses mapped by `middleware/errorHandler.ts`. Tests: vitest, mocking `../config/firebase.js` with an in-memory Firestore (see `src/__tests__/userRepository.test.ts`).
- **Backend commands** (run in `backend/`): `npm run typecheck`, `npm run lint`, `npm test` (vitest), `npm run build`.
- **Flutter** (`frontend/`): Riverpod + go_router + Dio `ApiClient` (unwraps `{ success, data }`) + firebase_auth. Layering `models / services / repositories / providers / features/*`. UI → providers → repositories → services; repositories return `Result<T>`. Theme = Midnight Energy via `AppColors`/`AppShadows`/`AppSpacing`. Tests under `frontend/test/*_test.dart` (`flutter_test`; package import `package:fittrack/...`).
- **Flutter commands** (run in `frontend/`): `flutter analyze`, `flutter test`, `flutter build apk --debug`.
- **Security invariants (non-negotiable):** role is server-trusted (resolved on `verify`, never from client); all trainer→student reads go through the backend Admin SDK after `assertTrainerOwnsStudent`; `trainers`/`referralCodes`/`trainerLinks` are never client-readable/writable; student data under `users/{uid}/**` stays owner-only; progress-photo bytes served only via short-lived signed URLs.

---

## A. Nutrition Add-Food bug (independent; do first — smallest, highest-value, unblocks a clean baseline)

- [ ] 1. Backend: carry `mealId`/`mealName` through the nutrition read path (root-cause fix, design §7.1–7.2).
      Add `mealId?: string | null;` and `mealName?: string | null;` to the `FoodLogEntry` interface in `backend/src/utils/nutritionCalc.ts` (keep `groupByMeal` keyed by `mealType`). In `backend/src/services/nutritionService.ts` `getDay()` row mapping, add `mealId: (r.mealId as string) ?? null, mealName: (r.mealName as string) ?? null` so the top-level `entries[]` retains the association. No write-path change (write already persists these via `foodLogUpsertSchema`).
      Files: `backend/src/utils/nutritionCalc.ts`, `backend/src/services/nutritionService.ts`
      Verify: `npm test` in `backend/` — add/extend a nutritionService test (step 3) asserting a logged entry with `mealType:'custom', mealId:'m1', mealName:'Pre-workout'` round-trips through `getDay` with both fields intact. `npm run typecheck` passes.

- [ ] 2. Backend: client-supplied-id idempotency for Add Food (design §7.4).
      Add `clientId: z.string().min(1).max(200).optional()` to `foodLogUpsertSchema` in `backend/src/validators/nutritionValidators.ts`. In `nutritionService.addFood`, use `const id = input.clientId ?? nutritionRepository.newLogId(uid);` then `setLog(uid, id, ...)` (strip `clientId` from the stored payload — it only drives the doc id). `setLog` already uses `{ merge: true }`, so a re-tap with the same id updates one record instead of duplicating.
      Files: `backend/src/validators/nutritionValidators.ts`, `backend/src/services/nutritionService.ts`
      Verify: `npm test` in `backend/` — add a test calling `addFood` twice with the same `clientId` and asserting a single `foodLogs` doc / no doubled totals (step 3). `npm run typecheck` passes.

- [ ] 3. Backend: nutrition round-trip + idempotency tests.
      Add `backend/src/__tests__/nutritionService.test.ts` following the `userRepository.test.ts` pattern (mock `../config/firebase.js` with an in-memory Firestore). Cover: (a) `getDay` preserves `mealId`/`mealName`; (b) totals computed correctly incl. fiber/sugar/sodium; (c) duplicate `clientId` add writes once.
      Files: `backend/src/__tests__/nutritionService.test.ts`
      Verify: `npm test` in `backend/` — new tests pass; existing `nutritionCalc.test.ts` still passes.

- [ ] 4. Flutter: send a stable `clientId` per sheet open and keep invalidate-before-pop (design §7.4–7.5).
      In `frontend/lib/features/nutrition/presentation/widgets/add_food_sheet.dart`, generate one stable id in `initState` (e.g. `DateTime.now().microsecondsSinceEpoch.toString()`), capture `dateKey` at open, and pass the id into the built `FoodLogEntry`. In `frontend/lib/models/food.dart` add an optional `clientId` field and emit it in `toJson()` when non-null. Confirm `_add()` keeps `ref.invalidate(nutritionDayProvider)` before `Navigator.pop()`, the `_saving` guard, and SnackBar-on-failure.
      Files: `frontend/lib/models/food.dart`, `frontend/lib/features/nutrition/presentation/widgets/add_food_sheet.dart`
      Verify: `flutter analyze` clean; `flutter test` passes (existing `food_model_test.dart` plus the new widget test in step 5).

- [ ] 5. Flutter: Add-Food widget test (named custom meal appears + macro summary updates).
      Add `frontend/test/add_food_sheet_test.dart` overriding `nutritionRepositoryProvider`/`nutritionDayProvider` with fakes: add a food to a named custom meal, refresh, assert it renders in that meal's section via `NutritionDay.entriesForCustomMeal` and the daily totals include it.
      Files: `frontend/test/add_food_sheet_test.dart`
      Verify: `flutter test` in `frontend/` — new test passes.

---

## B. Backend data model, role, and seed (foundation for the trainer layer)

- [ ] 6. Backend: role resolution service + `getRole`, server-trusted role on verify (design §2.4–2.5, §5.1).
      Add `backend/src/services/roleService.ts` with `resolveRole(uid)` = `users/{uid}.role ?? (trainers/{uid} exists ? 'trainer' : 'student')`, purely computed (never writes `role` back). Add `UserRepository.getRole(uid)` to `backend/src/repositories/userRepository.ts` and confirm `ensureAccount` never writes/clears `role`. Extend `authService.verifyAndSync` to inject the resolved `role` into the returned account object (no `authController.verify` change needed — it already returns `{ account }`).
      Files: `backend/src/services/roleService.ts`, `backend/src/repositories/userRepository.ts`, `backend/src/services/authService.ts`
      Verify: `npm test` in `backend/` — new `roleService` tests (step 13) pass; `userRepository.test.ts` still passes (no-write-on-unchanged-reopen preserved). `npm run typecheck` passes.

- [ ] 7. Backend: trainer + link repositories and validators (design §3, §5.5).
      Add `backend/src/repositories/trainerRepository.ts` (`trainers/{trainerId}` get/upsert; `referralCodes/{codeLower}` lookup; live `countActiveStudents(trainerId)` via `trainerLinks where trainerId==, status=='active'`). Add `backend/src/repositories/trainerLinkRepository.ts` (`trainerLinks/{studentUid}` get/create-in-transaction; `listByTrainer(trainerId)`). Add `backend/src/validators/trainerValidators.ts` with `linkTrainerSchema`, `studentUidParamsSchema`, `dateQuerySchema` (reuse `commonSchemas.dateKey.optional()`), `historyQuerySchema`, `sharingPatchSchema`.
      Files: `backend/src/repositories/trainerRepository.ts`, `backend/src/repositories/trainerLinkRepository.ts`, `backend/src/validators/trainerValidators.ts`
      Verify: `npm run typecheck` + `npm run lint` in `backend/` pass. (Behavior covered by step 9/13 tests.)

- [ ] 8. Backend: extend profile schema with `trainerId` + `shareProgressWithTrainer` (design §3.2).
      Add optional `trainerId: z.string().nullable().optional()` and `shareProgressWithTrainer: z.boolean().optional()` to the profile upsert schema in `backend/src/validators/profileValidators.ts` so existing upserts stay valid and onboarding is unaffected.
      Files: `backend/src/validators/profileValidators.ts`
      Verify: `npm run typecheck` in `backend/` passes; `npm test` existing profile/validator tests still pass.

- [ ] 9. Backend: trainer seed script + `seed:trainer` npm script (design §2.4).
      Add `backend/src/scripts/seedTrainer.ts` (mirror `seedExercises.ts`): read `SEED_TRAINER_UID` from env, idempotently upsert `users/{uid}.role='trainer'`, `trainers/{uid}` (`name:'Dream Physics'`, `referralCode:'dreamphysics'`, `referralCodeLower:'dreamphysics'`, `totalStudents:0`, `status:'active'`, `photoUrl:null`, timestamps), and `referralCodes/dreamphysics → { trainerId: uid }`. Add `"seed:trainer": "tsx src/scripts/seedTrainer.ts"` to `backend/package.json`. Guard on `hasFirebaseCredentials()` like `seedExercises.ts`.
      Files: `backend/src/scripts/seedTrainer.ts`, `backend/package.json`
      Verify: `npm run typecheck` in `backend/` passes; `npm run build` compiles. (Script execution requires real credentials and is run manually, not in CI.)

---

## C. Backend trainer endpoints + authorization

- [ ] 10. Backend: role middleware + ownership guard (design §5.1, §5.6).
      Add `backend/src/middleware/role.ts`: `requireTrainer` (after `authenticate`, resolves role, throws `403 ForbiddenError` if not `trainer`) and `assertTrainerOwnsStudent(trainerId, studentUid)` (reads `trainerLinks/{studentUid}`; throws `404 NotFoundError` if missing, trainerId mismatch, or `status !== 'active'`).
      Files: `backend/src/middleware/role.ts`
      Verify: `npm test` in `backend/` — authorization-matrix tests (step 13) pass. `npm run typecheck` passes.

- [ ] 11. Backend: registration linking endpoint (design §5.2).
      Add `POST /api/auth/link-trainer` to `backend/src/routes/auth.routes.ts` (authenticated; `validate({ body: linkTrainerSchema })`) with `authController.linkTrainer` → `trainerService.linkStudent(uid, code)`: normalize code; lookup `referralCodes/{code}`; if absent return `200 { linked:false, reason:'invalid_code' }`; else run a Firestore transaction creating `trainerLinks/{uid}` (`status:'active'`), mirror `profile/data.trainerId`, increment `trainers/{trainerId}.totalStudents` (idempotent — no double count if link exists); a `trainer`-role account → `409 ConflictError`. Return `{ linked:true, trainerId, trainerName }`.
      Files: `backend/src/routes/auth.routes.ts`, `backend/src/controllers/authController.ts`, `backend/src/services/trainerService.ts`
      Verify: `npm test` in `backend/` — link tests (valid/invalid/duplicate/trainer-as-student) in step 13 pass. `npm run typecheck` passes.

- [ ] 12. Backend: trainer feature slice (routes/controller/service) + mount (design §5.3, §5.4).
      Add `backend/src/routes/trainer.routes.ts` (all behind `authenticate` + `requireTrainer`), `backend/src/controllers/trainerController.ts`, and extend `backend/src/services/trainerService.ts` for: `GET /api/trainer/profile` (live active-student count), `GET /api/trainer/students` (summaries), and per-student `overview`, `workouts?date=`, `workouts/history?limit=`, `nutrition?date=` (reuse `nutritionService.getDay(studentUid, date)`), `water?date=`, `progress`, `photos` (gated by `shareProgressWithTrainer`, signed URLs via `progressPhotoService`). Every per-student handler calls `assertTrainerOwnsStudent(req.uid, studentUid)` first. Add student-facing `GET /api/student/trainer` (non-active link → `{ trainer:null }`) and `PATCH /api/student/trainer/sharing`. Mount `/api/trainer` and `/api/student` in `backend/src/app.ts`. Extend `progressPhotoService` with an authorized signed-URL method (per-photo `url:null` on signing failure; Storage rules unchanged).
      Files: `backend/src/routes/trainer.routes.ts`, `backend/src/controllers/trainerController.ts`, `backend/src/services/trainerService.ts`, `backend/src/routes/student.routes.ts`, `backend/src/controllers/studentController.ts`, `backend/src/services/progressPhotoService.ts`, `backend/src/app.ts`
      Verify: `npm test` in `backend/` — route authorization-matrix tests (step 13) pass. `npm run typecheck` + `npm run lint` pass.

- [ ] 13. Backend: trainer/role/link/authorization + auth tests.
      Add `backend/src/__tests__/trainerService.test.ts`, `roleService.test.ts`, and route-level tests (vitest, mocked Firestore). Cover: `resolveRole` (present / absent+trainer / absent→student); `assertTrainerOwnsStudent` (owned→ok, other trainer→404, missing→404, inactive→404); `link-trainer` (valid→link+increment once, invalid→soft 200, duplicate→idempotent, trainer-as-student→409); referral normalization; registration no-code (no link) / valid-code / invalid-code; trainer login role resolution; existing-user login; authorization matrix 401/403/404/200; trainer-reads-student-nutrition round-trip.
      Files: `backend/src/__tests__/trainerService.test.ts`, `backend/src/__tests__/roleService.test.ts`, `backend/src/__tests__/trainerRoutes.test.ts`
      Verify: `npm test` in `backend/` — all new tests pass with the full suite green.

---

## D. Firestore rules & indexes

- [ ] 14. Firebase: fail-closed rules + composite index (design §3.6, §4).
      In `c:\Users\cheta\OneDrive\Desktop\fit\firebase\firestore.rules`, add explicit client deny for `trainers/{trainerId}`, `referralCodes/{code}`, `trainerLinks/{studentUid}` (`allow read, write: if false;`) above the default-deny, with a comment that these are self-documenting guards redundant with the trailing catch-all (per design-review NIT 1). Leave `users/{uid}/**` owner-only and `storage.rules` unchanged. Add one composite index to `firebase/firestore.indexes.json`: collection `trainerLinks`, fields `trainerId ASC, status ASC, updatedAt DESC`.
      Files: `c:\Users\cheta\OneDrive\Desktop\fit\firebase\firestore.rules`, `c:\Users\cheta\OneDrive\Desktop\fit\firebase\firestore.indexes.json`
      Verify: `firestore.indexes.json` parses as valid JSON. Flag `firestore.rules` (unversioned) for user review. (Emulator rule tests are integration-only; not wired into CI per design §4/§8.)

---

## E. Flutter role-based routing & trainer UI

- [ ] 15. Flutter: `role` on `AccountInfo` + role branch before the onboarding gate (design §6.1).
      Add `final String role;` to `AccountInfo` in `frontend/lib/features/auth/data/auth_repository.dart`, parsed as `account['role'] as String? ?? 'student'`. Add `trainerHome = '/trainer'` to `frontend/lib/core/routing/route_names.dart`. In `frontend/lib/core/routing/app_router.dart`, insert the role branch immediately after `accountAsync` loads successfully and BEFORE `final onboarded = ...`: if `account?.role == 'trainer'`, route to `Routes.trainerHome` (stay if already there); register `GoRoute(path: Routes.trainerHome, builder: (_, __) => const TrainerShell())`. Student flow unchanged.
      Files: `frontend/lib/features/auth/data/auth_repository.dart`, `frontend/lib/core/routing/route_names.dart`, `frontend/lib/core/routing/app_router.dart`
      Verify: `flutter analyze` clean; `flutter test` — router test (step 19) routes `role:trainer`→trainer shell, `role:student`/legacy→`HomeShell`. `flutter build apk --debug` succeeds.

- [ ] 16. Flutter: trainer data layer + models + providers (design §6.5).
      Add `frontend/lib/features/trainer/data/trainer_repository.dart` wrapping `ApiClient` for all `/api/trainer/*` and `/api/student/trainer` calls (returning `Result<T>`), typed models (`TrainerProfile`, `TrainerStudentSummary`, `StudentOverview`; reuse `NutritionDay`/`FoodLogEntry` where shapes match), and `frontend/lib/features/trainer/providers/trainer_providers.dart` (`FutureProvider` / `FutureProvider.family` keyed by `studentUid`[+date]).
      Files: `frontend/lib/features/trainer/data/trainer_repository.dart`, `frontend/lib/features/trainer/data/trainer_models.dart`, `frontend/lib/features/trainer/providers/trainer_providers.dart`
      Verify: `flutter analyze` clean.

- [ ] 17. Flutter: trainer shell + screens on the Midnight Energy theme (design §6.2–6.3).
      Add `frontend/lib/features/trainer/presentation/trainer_shell.dart` — its own `Scaffold` + `IndexedStack` + a Midnight-styled nav bar (reuse `AppColors`/`AppShadows`/`AppSpacing`, mirroring `_MidnightNavBar`) with tabs Dashboard, Students, Progress, Profile. Add `trainer_dashboard_screen.dart` (overview cards + student list with client-side search/sort/filter All/Active/Inactive/Recently active/Goal/Progress; cards show photo, name, goal, current weight, progress summary, last workout, today's activity, nutrition status, overall progress), `trainer_students_screen.dart`, `student_detail_screen.dart` (`TabBar`/`TabBarView`: Overview / Workout / Workout History / Nutrition / Water / Progress / Progress Photos, each a `FutureProvider.family`; reuse existing chart widgets like `MacroRing`), `trainer_progress_screen.dart`, `trainer_profile_screen.dart`. The student `HomeShell` is NOT modified. Handle empty/loading/error states (zero students, no data for a date, sharing-off, 404-after-unlink).
      Files: `frontend/lib/features/trainer/presentation/trainer_shell.dart`, `.../trainer_dashboard_screen.dart`, `.../trainer_students_screen.dart`, `.../student_detail_screen.dart`, `.../trainer_progress_screen.dart`, `.../trainer_profile_screen.dart`
      Verify: `flutter analyze` clean; `flutter test` — dashboard/student-detail render tests with empty/loading states (step 19) pass; `flutter build apk --debug` succeeds.

---

## F. Flutter student-side additions (non-breaking)

- [ ] 18. Flutter: referral field, link-after-register, invalid-code surfacing, Trainer Login, My Trainer card (design §6.4).
      In `frontend/lib/features/auth/data/auth_repository.dart` add `linkTrainer(code)` hitting `POST /api/auth/link-trainer` and a `linkTrainerResultProvider` (`StateProvider<LinkTrainerResult?>`) written via a container/notifier that outlives the register screen (design-review NIT 4). In `register_screen.dart` add ONE optional `TextFormField` with placeholder exactly `Enter trainer referral code (optional)` (no blocking validator); if non-empty, call `linkTrainer` right after a successful `register()`. In `verify_email_screen.dart` read `linkTrainerResultProvider` and surface: invalid → "Invalid trainer referral code" (non-blocking, still a student); linked → "Connected to Dream Physics"; clear once shown. In `login_screen.dart` add a "Trainer Login" secondary action that uses the same Firebase email/password sign-in (role resolved by backend). In `profile_screen.dart` add a "My Trainer" card (name/photo/status from `GET /api/student/trainer`) + the `shareProgressWithTrainer` toggle.
      Files: `frontend/lib/features/auth/data/auth_repository.dart`, `frontend/lib/features/auth/presentation/register_screen.dart`, `frontend/lib/features/auth/presentation/verify_email_screen.dart`, `frontend/lib/features/auth/presentation/login_screen.dart`, `frontend/lib/features/profile/presentation/profile_screen.dart`
      Verify: `flutter analyze` clean; `flutter test` — register-with-invalid-code test (step 19) shows the message and lands as student; `flutter build apk --debug` succeeds.

- [ ] 19. Flutter: routing + trainer-UI + register tests.
      Add `frontend/test/trainer_routing_test.dart` (role trainer→trainer shell, student/legacy→`HomeShell`), `frontend/test/trainer_dashboard_test.dart` and `frontend/test/student_detail_test.dart` (render with empty/loading/data via overridden providers), and `frontend/test/register_referral_test.dart` (invalid code message + still-student). Follow the existing `*_test.dart` style.
      Files: `frontend/test/trainer_routing_test.dart`, `frontend/test/trainer_dashboard_test.dart`, `frontend/test/student_detail_test.dart`, `frontend/test/register_referral_test.dart`
      Verify: `flutter test` in `frontend/` — all new tests pass.

---

## G. Full verification (integration seam check)

- [ ] 20. Run the complete verification matrix and fix any failures.
      Backend (in `backend/`): `npm run typecheck`, `npm run lint`, `npm test`. Flutter (in `frontend/`): `flutter analyze`, `flutter test`, `flutter build apk --debug`. Confirm no existing student feature regressed (Home/Workout/Nutrition/AI/Profile/Water/Routine/Alarm/Progress/History/Analytics/Notifications) and the Midnight Energy shell is unchanged. Flag `firebase/firestore.rules` and the new composite index for user review.
      Files: (whole repo)
      Verify: all six commands succeed with no new errors; acceptance criteria §9.1–9.7 in the design satisfied.

---

## Notes / assumptions

- Git is on branch `feat/trainer-student-2way`; coder steps commit locally and never push. (The planning shell could not run git against the OneDrive path; coder steps run where git works.)
- Signed-URL generation and real Firestore/Storage are integration-only (key-bearing creds); do NOT wire CI tests asserting real signed URLs (design §2.3).
- The nutrition fix (Section A) is independent of the trainer layer and is sequenced first so the baseline is green before the larger additive work.
