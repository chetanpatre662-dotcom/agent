# Implementation Plan — FitTrack AI Trainer/User Role & Session Fix

Scope: fill the specific gaps the audit identifies WITHOUT rebuilding the already-correct role/session system. The one genuine behavioral change is replacing the immediate trainer auto-link with a PENDING request + Approve/Reject flow. Everything else (role resolution, Trainer Code generation, real student-detail reads, backend ownership, fail-closed rules, backend-authoritative startup) is already DONE and must stay green.

All paths are absolute.

- BACKEND (worktree): `C:\Users\cheta\OneDrive\Desktop\fit\backend\.worktrees\trainer-role-session-fix` (code under `src\`), on branch `feat/trainer-role-session-fix` branched off `feat/trainer-student-2way`. All backend edits happen here.
- FRONTEND (in-place, NO worktree): `C:\Users\cheta\OneDrive\Desktop\fit\frontend` (code under `lib\`, tests under `test\`), on branch `feat/trainer-role-session-fix` branched off `feat/trainer-student-2way`. All frontend edits and `flutter analyze`/`flutter test` happen in this live dir — the real app lives on disk here as mostly-UNTRACKED baseline.
- The `firebase\` dir is NOT versioned and must NEVER be committed.

Git staging rules (both repos): stage ONLY the specific files this task creates/modifies. NEVER `git add -A` / `git add .` — the frontend baseline (pubspec.yaml, lib/*, test/*, etc.) is intentionally untracked and must be left alone. No force-push, no git config changes. Push/merge is left to the user.

---

## ITEM 0 — base branch corrected (DONE; context for the implementer)

The setup step originally created the worktree branches off `main`/`master`, which predate the entire trainer foundation (that code lives only on `feat/trainer-student-2way`). This has been FIXED before implementation:

- Backend: the `.worktrees/trainer-role-session-fix` created off `main` was removed; the branch was recreated off `feat/trainer-student-2way` and re-added as the worktree. Verified `trainerService.ts`, `trainerLinkRepository.ts`, `roleService.ts`, `studentService.ts`, `trainer.routes.ts`/`student.routes.ts`, `trainerController.ts`/`studentController.ts`, `trainerValidators.ts`, `middleware/role.ts`, `models/domain.ts` all PRESENT. Baseline green: `npm run typecheck` + `npm test` = 21 files / 149 tests pass (incl. `trainerService.test.ts` 25, `trainerRoutes.test.ts` 15).
- Frontend: NOT a worktree. The frontend worktree was removed and the branch `feat/trainer-role-session-fix` was created in-place in the live `C:\Users\cheta\OneDrive\Desktop\fit\frontend` dir off `feat/trainer-student-2way`. The untracked app baseline (pubspec.yaml, lib/main.dart, lib/features/*, test/*) is intact on disk. Run all Flutter commands here.

Final merge/finalize step MUST rebase/merge backend `feat/trainer-role-session-fix` onto `feat/trainer-student-2way` (NOT `main`), commit scoped frontend files on the in-place frontend branch (or, as a flagged fallback, onto `feat/trainer-student-2way`), never touch the untracked frontend baseline or git config, no force-push, no `git add -A`, and leave push/merge to the user. `firebase/` stays uncommitted and is flagged for the user.

Paths below use the corrected roots above.

---

## Backend — status enum + pending/approve/reject (the core change)

- [ ] 1. Add a central trainer-link status enum/constants to the shared domain model.
      Add `export const TRAINER_LINK_STATUSES = ['pending','active','rejected','inactive'] as const;` and `export type TrainerLinkStatus = (typeof TRAINER_LINK_STATUSES)[number];` to `backend\src\models\domain.ts`. Document that `'active'` IS the approved state (reused to avoid churning `listByTrainer`/`countActiveStudents`/`assertTrainerOwnsStudent`), `'pending'` = awaiting trainer approval, `'rejected'` = declined. Design rationale (record inline): reuse the single `trainerLinks/{studentUid}` doc with a status enum rather than a new collection, because the single-trainer invariant (one doc per student) is already enforced and keeps existing ownership/count queries intact.
      Files: `backend\src\models\domain.ts`
      Verify: `cd backend\.worktrees\trainer-role-session-fix && npm run typecheck` passes.

- [ ] 2. Extend `trainerLinkRepository.ts` with pending-request read helpers.
      Add `listRequestsByTrainer(trainerId)` = `where('trainerId','==',t).where('status','==','pending')` then in-memory sort by `updatedAt` desc (mirror the existing `listByTrainer` shape; reuses the existing `(trainerId ASC, status ASC, updatedAt DESC)` composite index — NO new index needed since we order by `updatedAt`). Keep the existing `get`, `listByTrainer` (status=='active'), `doc`, `fieldValue` members unchanged.
      Files: `backend\src\repositories\trainerLinkRepository.ts`
      Verify: `npm run typecheck` passes.

- [ ] 3. Add request/approve/reject service methods to `trainerService.ts` (do NOT remove `linkStudent`; the approve path reuses its transaction shape).
      Add `requestTrainer(studentUid, code)`: validate EXACTLY like `linkStudent` (resolve `referralCodes/{codeLower}` → active trainer; unknown/not-a-trainer → soft `{ ok:false, reason:'invalid_code' }`; caller role 'trainer' → `ConflictError`/409). Then: if an existing link to the SAME trainer is already `pending` or `active` → idempotent soft success (`{ ok:true, status:<existing> }`, no write). If an `active` link to a DIFFERENT trainer exists → `{ ok:false, reason:'already_linked', currentTrainer, requestedTrainer }` (single-trainer invariant, no silent overwrite — a switch is a NEW pending request, never an auto-switch). Otherwise write `trainerLinks/{studentUid}` with `status:'pending'`, `trainerId`, `createdAt`(if new)/`updatedAt` — and critically do NOT mirror `profile.trainerId` and do NOT increment `totalStudents`.
      Add `approveRequest(trainerId, studentUid)`: read the link; require it exists, `trainerId == caller`, and `status === 'pending'` (else 404 via the pending-aware check from item 4). In one transaction set `status:'active'`, mirror `users/{studentUid}/profile/data.trainerId` AND new field `trainerStatus:'approved'`, and `+1 totalStudents` — reuse the `createStudentLink` transaction pattern (guarded increment).
      Add `rejectRequest(trainerId, studentUid)`: require pending + ownership; set `status:'rejected'` (no profile mirror, no counter change).
      Add `listRequestsForTrainer(trainerId)`: call `trainerLinkRepository.listRequestsByTrainer`, then hydrate each with the student's name + photo from `profileRepository.get(studentUid)` (same enrichment style as `listStudents`, but name+photo only).
      Rename `LinkResult.linked` usages are NOT required — keep `linked` for the existing link path; the new methods return their own `{ ok, reason?, status? }` envelope.
      Files: `backend\src\services\trainerService.ts`
      Verify: `npm run typecheck` passes.

- [ ] 4. Add a pending-aware ownership check to `middleware/role.ts` WITHOUT loosening `assertTrainerOwnsStudent`.
      Keep `assertTrainerOwnsStudent` requiring `status==='active'` verbatim (a pending link must grant NO data access). Add a separate `assertTrainerOwnsPendingRequest(trainerId, studentUid)` that reads `trainerLinks/{studentUid}` and throws `NotFoundError` (404, not 403 — same anti-enumeration reasoning) unless the link exists, `trainerId === caller`, and `status==='pending'`. Used only by approve/reject.
      Files: `backend\src\middleware\role.ts`
      Verify: `npm run typecheck` passes.

- [ ] 5. Add Zod schemas for the new endpoints in `trainerValidators.ts`.
      Add `requestsStudentParamsSchema` = reuse `studentUidParamsSchema` (studentUid 1..128) for the `:studentUid` approve/reject params (or re-export it). No request body is needed for approve/reject (trainer identity comes from the token). Keep `linkTrainerSchema` as-is (still used by the now-request-creating connect/link endpoints — body stays `{ referralCode, confirmSwitch? }`). Export any new inferred types.
      Files: `backend\src\validators\trainerValidators.ts`
      Verify: `npm run typecheck` passes.

- [ ] 6. Add trainer-only Requests endpoints in `trainer.routes.ts` + handlers in `trainerController.ts`.
      Routes (after the existing `router.use(authenticate); router.use(requireTrainer);`): `GET /requests` → `listRequests`; `POST /requests/:studentUid/approve` (validate params) → `approveRequest`; `POST /requests/:studentUid/reject` (validate params) → `rejectRequest`. Controllers call `requireUid(req)` for trainerId, call the item-3 service methods, and return the standard `ok(res, {...})` envelope (`{ requests }` for the list; `{ ok:true, studentUid, status }` for approve/reject). Approve/reject MUST call `assertTrainerOwnsPendingRequest` first (via the service method, which already does).
      Files: `backend\src\routes\trainer.routes.ts`, `backend\src\controllers\trainerController.ts`
      Verify: `npm run typecheck` passes.

- [ ] 7. Switch the two immediate-link entry points to request semantics (no active link on connect).
      `studentController.connectTrainer` (`POST /api/student/connect-trainer`) and `authController.linkTrainer` (`POST /api/auth/link-trainer`): call `trainerService.requestTrainer(uid, referralCode)` instead of `linkStudent`. Preserve soft-fail-on-invalid-code (unknown code → `{ ok:false, reason:'invalid_code' }`, HTTP 200 — never error out registration). A valid code now creates a PENDING request, not an active link. Keep the switch case coherent: a request to a different trainer while actively linked returns `already_linked` (the student must get removed/approved elsewhere; no auto-switch). Leave `linkStudent` in the service for now (approve reuses its tx shape) but it is no longer reachable from these two routes.
      Files: `backend\src\controllers\studentController.ts`, `backend\src\controllers\authController.ts`
      Verify: `npm run typecheck` passes.

- [ ] 8. (firebase — DO NOT COMMIT) Confirm no new index is required; add a self-doc rule only if needed.
      The existing `trainerLinks (trainerId ASC, status ASC, updatedAt DESC)` composite in `C:\Users\cheta\OneDrive\Desktop\fit\firebase\firestore.indexes.json` already covers the `status=='pending'` + orderBy `updatedAt` query — NO index change needed. `trainerLinks` is already backend-only in `firestore.rules`; the new `trainerStatus` mirror lives under `users/{uid}/profile/data` (already owner-readable). So no rules change is required either. If any firebase file IS edited, it must NOT be committed (it has no worktree) — flag it to the user.
      Files: none expected (verification only); if touched: `C:\Users\cheta\OneDrive\Desktop\fit\firebase\firestore.indexes.json` (NEVER commit)
      Verify: visually confirm the composite index exists; no code change to verify.

> FRONTEND path note: all frontend files below are under the live dir `C:\Users\cheta\OneDrive\Desktop\fit\frontend\lib\...` and `...\test\...`. Run `flutter analyze` / `flutter test` from `C:\Users\cheta\OneDrive\Desktop\fit\frontend`. Stage only the scoped files you change; never `git add -A`.

## Backend — tests (lockstep with the behavioral change)

- [ ] 9. Update/extend backend tests to the NEW approval behavior (do NOT weaken behavior to satisfy old auto-link assertions).
      `trainerService.test.ts`: change assertions that expected `status:'active'` immediately after connect/link to expect `status:'pending'` with NO `profile.trainerId` mirror and NO `totalStudents` increment; add cases for `requestTrainer` (invalid_code soft-fail, trainer-as-student 409, duplicate pending idempotent, already_linked-to-different-trainer), `approveRequest` (pending→active + mirror trainerId + trainerStatus:'approved' + count +1; non-pending/non-owner → 404), `rejectRequest` (pending→rejected, no mirror/no count), `listRequestsForTrainer` (only pending, enriched with name+photo).
      `trainerRoutes.test.ts`: add `GET /api/trainer/requests`, `POST /api/trainer/requests/:studentUid/approve`, `.../reject` incl. 404 on wrong-trainer/non-pending; confirm `assertTrainerOwnsStudent` still 404s for a pending (not-yet-approved) link on the per-student data routes.
      Keep `roleService.test.ts`, `userRepository.test.ts`, `referralCode.test.ts`, `middleware.test.ts` and all unrelated suites GREEN (no behavior change there).
      Files: `backend\src\__tests__\trainerService.test.ts`, `backend\src\__tests__\trainerRoutes.test.ts`
      Verify: `cd backend\.worktrees\trainer-role-session-fix && npm run lint && npm run typecheck && npm test` — all suites pass.

## Frontend — Requests UI + request-not-link UX

- [ ] 10. Add request models + repo methods + providers for the trainer Requests flow.
      Models: in `lib\features\trainer\data\trainer_models.dart` add a `TrainerRequest` model (studentUid, name, photoUrl, requestedAt) parsing the real `{ requests: [...] }` payload — no mock data. Repo: in `lib\features\trainer\data\trainer_repository.dart` add `getRequests()` → `GET /api/trainer/requests` (returns `List<TrainerRequest>`), `approveRequest(studentUid)` → `POST /api/trainer/requests/:uid/approve`, `rejectRequest(studentUid)` → `POST /api/trainer/requests/:uid/reject`, each returning a `Result` and never throwing into UI (match the existing method style). Providers: in `lib\features\trainer\providers\trainer_providers.dart` add `trainerRequestsProvider` (FutureProvider) wired to `getRequests()`.
      Files: `lib\features\trainer\data\trainer_models.dart`, `lib\features\trainer\data\trainer_repository.dart`, `lib\features\trainer\providers\trainer_providers.dart`
      Verify: (run in live frontend dir) `flutter analyze` clean for these files.

- [ ] 11. Make the trainer Students screen a tabbed Students | Requests view with Approve/Reject.
      Convert `TrainerStudentsScreen` to a `DefaultTabController`/`TabBar` with two tabs. Students tab = the existing searchable list. Requests tab = `trainerRequestsProvider` list; each row shows student name + `[Approve]` `[Reject]`; approve/reject call the repo methods then `ref.invalidate(trainerRequestsProvider)` AND `ref.invalidate(trainerStudentsProvider)` (so an approved student moves into Students). Fix the now-wrong empty-state copy ("Students who sign up with your referral code appear here." → approval-aware wording, e.g. "Approved students appear here. Check the Requests tab for pending requests.").
      Files: `lib\features\trainer\presentation\trainer_students_screen.dart`
      Verify: `flutter analyze` clean; the Requests-tab widget test in item 15 passes.

- [ ] 12. Slim the Students-tab card to NAME + AVATAR only (keep tap → StudentDetailScreen unchanged).
      In `lib\features\trainer\presentation\widgets\trainer_widgets.dart`, change `StudentSummaryCard` to render only avatar + name (drop goal/weight/activity from the initial card). Tapping still opens `StudentDetailScreen` (real data, unchanged). Keep the backend `listStudents` payload as-is (card just renders less).
      Files: `lib\features\trainer\presentation\widgets\trainer_widgets.dart`
      Verify: `flutter analyze` clean; `trainer_dashboard_test.dart` still passes (update it in lockstep if it asserts the dropped fields).

- [ ] 13. Change student connect flow from link-immediately to SEND REQUEST (Profile + Register), and surface read-only trainerStatus.
      `lib\features\profile\presentation\profile_screen.dart`: the "Add Trainer"/connect sheet now sends a request; on success show "Request sent — waiting for trainer approval." On the "My Trainer" card, surface `trainerStatus` (pending/approved) read-only: when pending show "Pending approval"; when approved show the connected Trainer Code read-only; when no link show "Add Trainer". Parse `trainerStatus`/`associationStatus` from the `MyTrainer` model (extend `trainer_models.dart` if the field isn't parsed yet).
      `lib\features\auth\presentation\register_screen.dart`: register-time code entry creates a pending request (same soft-fail: an invalid/absent code never blocks registration); after submit, messaging reflects "request sent" rather than "connected".
      Files: `lib\features\profile\presentation\profile_screen.dart`, `lib\features\auth\presentation\register_screen.dart`, `lib\features\trainer\data\trainer_models.dart` (if `trainerStatus` parsing needed)
      Verify: `flutter analyze` clean; `connect_trainer_test.dart` and `register_referral_test.dart` updated in lockstep (item 15) pass.

## Frontend — UI rename "Referral Code" → "Trainer Code" (user-visible strings only)

- [ ] 14. Rename user-visible "Referral Code"/"referral" strings to "Trainer Code"; keep Dart identifiers, widget keys, and backend field names unchanged.
      Do NOT rename `referralCode` identifiers, `trainerReferralCodeProvider`, `ReferralCodeInfo`, or widget keys `connectReferralField`/`customReferralField` (renaming a key requires updating its test in lockstep — avoid unless necessary). Exact string locations (from audit; verify line numbers before editing):
      - `lib\features\trainer\presentation\trainer_profile_screen.dart`: `'YOUR REFERRAL CODE'` → `'MY TRAINER CODE'`; `'Referral code copied'` → `'Trainer code copied'`; sheet title `'Change referral code'` and labels → "Trainer Code" wording; present the card as "My Trainer Code / <code> / [Copy]" and show `[Generate Trainer Code]` when there is no code.
      - `lib\features\profile\presentation\profile_screen.dart`: `'Invalid trainer referral code'` → `'Invalid Trainer Code'`; `'Referral Code: <code>'` → `'Trainer Code: <code>'`; connect-sheet copy/labels; `'Enter the referral code your trainer gave you.'` → Trainer Code wording. Confirm the student's connected code renders as read-only plain text (it already does).
      - `lib\features\auth\presentation\register_screen.dart`: label/hint `'Trainer referral code'` / `'Enter trainer referral code (optional)'` → "Trainer Code".
      - `lib\features\auth\presentation\verify_email_screen.dart`: invalid-code strings → "Trainer Code".
      - `lib\features\trainer\presentation\trainer_students_screen.dart`: covered by item 11's empty-state fix.
      Files: `trainer_profile_screen.dart`, `profile_screen.dart`, `register_screen.dart`, `verify_email_screen.dart` (all under `lib\features\...`)
      Verify: `flutter analyze` clean; `trainer_referral_manage_test.dart`, `connect_trainer_test.dart` updated in lockstep (item 15) pass.

## Frontend — SessionStore (hint only) + logout no-leak

- [ ] 15. Add a client-local SessionStore (hint only) and write it on login/verify-success.
      New file `lib\core\session\session_store.dart`: a thin class backed by the existing `sharedPreferencesProvider`, persisting ONLY `firebaseUid`, `email`, `role` (NEVER password/token), with `save({uid,email,role})`, `read()`, and `clear()`. Expose a `sessionStoreProvider`. Write on the `accountInfoProvider`/verify success path (and/or `authController.login` success). HARD CONSTRAINTS: it is a startup HINT / display aid ONLY — do NOT add any new async gate to the go_router redirect; the router MUST keep using Firebase session + backend `/api/auth/verify` as the authoritative gates and the role source; local role must NEVER override backend role (preserve the hardened reopen/redirect race fix); and it must introduce NO Firestore write on reopen (SharedPreferences only).
      Files: `lib\core\session\session_store.dart`, plus the verify/login success site (`lib\features\auth\...\account_provider.dart` or `auth_controller.dart` — confirm exact provider name before editing)
      Verify: `flutter analyze` clean; `auth_restore_test.dart` and `auth_repository_test.dart` still pass (reopen invariant intact).

- [ ] 16. Clear the SessionStore and prevent trainer-provider leakage on BOTH logout paths.
      On logout: call `SessionStore.clear()` AND the existing Firebase `signOut()`. Fix the provider-leak risk: trainer providers (`trainerProfileProvider`, `trainerStudentsProvider`, `trainerRequestsProvider`, the `student*Provider` families, `myTrainerProvider`, `trainerReferralCodeProvider`, `trainerTabProvider`) do not watch the current user; explicitly `ref.invalidate(...)` them on logout (or scope them to the signed-in uid) so Trainer A's students/Trainer Code cannot surface for a later-logged-in User B. Apply to BOTH paths: student `profile_screen._confirmLogout` and `trainer_profile_screen` logout → `authController.logout` / `auth_repository.logout`.
      Files: `lib\features\profile\presentation\profile_screen.dart`, `lib\features\trainer\presentation\trainer_profile_screen.dart`, `lib\features\auth\...\auth_controller.dart`, `lib\features\auth\data\auth_repository.dart`, `lib\features\trainer\providers\trainer_providers.dart`
      Verify: `flutter analyze` clean; the logout-no-leak extension to `trainer_routing_test.dart` (item 17) passes.

## Frontend — tests (lockstep)

- [ ] 17. Update/add frontend tests to the NEW approval + rename + session behavior.
      `connect_trainer_test.dart`: update to "send request" semantics + "Trainer Code" labels (not immediate connect). `register_referral_test.dart`: update register-time code entry to request semantics. `trainer_referral_manage_test.dart`: update to "Trainer Code" strings (keep `customReferralField` key unless renamed in lockstep). NEW `test\trainer_requests_test.dart`: Requests tab lists pending requests and Approve/Reject call the endpoints + invalidate providers. `trainer_routing_test.dart`: add a logout-no-leak case (Trainer A logout → User B login shows no Trainer A students/Trainer Code). Keep `account_type_selector_test.dart`, `student_detail_test.dart`, `trainer_dashboard_test.dart` (update only the slim-card assertions), `auth_restore_test.dart`, `auth_repository_test.dart`, `auth_error_mapper_test.dart`, `user_profile_test.dart`, `widget_test.dart` and all unrelated suites GREEN.
      Files: `test\connect_trainer_test.dart`, `test\register_referral_test.dart`, `test\trainer_referral_manage_test.dart`, `test\trainer_requests_test.dart` (new), `test\trainer_routing_test.dart`
      Verify: in `C:\Users\cheta\OneDrive\Desktop\fit\frontend` run `flutter analyze` and `flutter test` — all pass.

## Final verification (requirement 9 + task close-out)

- [ ] 18. Full-suite verification across both repos and manual walkthrough of the 13 spec cases.
      Backend: `cd backend\.worktrees\trainer-role-session-fix && npm run lint && npm run typecheck && npm test` all green. Frontend (in `C:\Users\cheta\OneDrive\Desktop\fit\frontend`): `flutter analyze` clean and `flutter test` green. Spot-check the behavioral spec cases: create user→role=user; create trainer→role=trainer+code; user cannot become trainer via Trainer Login; Add Trainer → valid code → pending request (not immediate link); trainer sees request → approve → student appears in My Students; reject handled; student sees connected Trainer Code read-only; trainer opens real Student Details; invalid/duplicate code handled soft; app reopen → correct role dashboard (no local-role override, no Firestore write); logout → login, no cross-account leak; Trainer A cannot read Trainer B's students (assertTrainerOwnsStudent still 404s).
      Files: none (verification only)
      Verify: both suites green; no regression in Workout/Nutrition/Water/Routine/AI Coach/Profile/Notifications/Auth/navigation.

---

## Notes carried for the implementer
- Keep backend field names `referralCode`/`referralCodeLower`/`referralCodeUpdatedAt` STABLE; the TRN- display is illustrative — do NOT change the code generator or invalidate existing codes.
- `'active'` == APPROVED is intentional reuse; `assertTrainerOwnsStudent` must keep requiring `status==='active'` so a pending link grants no data access.
- The behavioral change intentionally invalidates old auto-link test assertions — update tests to the new approval behavior; never weaken behavior to pass stale tests.
- Firebase config dir has no worktree and must never be committed.
