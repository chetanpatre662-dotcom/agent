# FitTrack AI — Trainer/User Role & Session Audit

READ-ONLY audit. No code changed. All paths absolute. The implementation workflow should be built directly from this report.

---

## 0. SUMMARY ANSWER (the single most important finding)

Most of the requested end state is **already implemented and compliant** from the previous "dynamic-multi-trainer-referral" and "trainer-student-2way" tasks. Specifically:

- Role is **server-trusted** (`roleService.resolveRole`), persisted in `users/{uid}.role`, returned by `/api/auth/verify`, and routing branches on it (`appRouterProvider`). A client can never set its own role.
- Registration already has a **User/Trainer selector**; "Trainer Login" on the login screen already **does NOT grant trainer** — it just submits the same email/password form.
- Trainer Code generation/uniqueness/persistence, the trainer "My Students"/detail screens reading **real** student data, backend ownership authorization (`assertTrainerOwnsStudent`), and fail-closed Firestore rules are all **DONE**.

The ONE genuine behavioral gap is **Requirement 3 (Connection + Approval)**. The current model **links immediately** on a valid code (`trainerLinks.status='active'`) with no pending request and no trainer Approve/Reject. The new requirement is: valid code → **PENDING request** → trainer approves/rejects → only then `trainerId`/`approved`. This is a MISSING feature touching backend service/repo/routes/validators, a new Firestore index, and new frontend UI (Requests tab + request-not-connect UX). It also drives label renames and the Students-tab "name + avatar only" tweak.

Secondary: Requirement 2 UI labels still say "Referral Code" (must become "Trainer Code"); Requirement 6 SharedPreferences `firebaseUid/email/role` persistence **does not exist** (startup today relies on Firebase persistence + backend verify, with no local session metadata written) — this is a MISSING-but-additive requirement.

---

## A. ROLE PERSISTENCE

**Storage/resolution — server-trusted, never client-supplied.**

- `backend/src/services/roleService.ts` — `RoleService.resolveRole(uid)`:
  1. `users/{uid}.role` if it is `'trainer'|'student'`;
  2. else `trainers/{uid}` doc exists → `'trainer'`;
  3. else `'student'` (legacy default). It **never writes** the role back. Type `AccountRole = 'student' | 'trainer'`.
- `backend/src/repositories/userRepository.ts` — `getRole(uid)` reads `users/{uid}.role` (string or null); `ensureAccount` creates the account WITHOUT a `role` field (so new plain accounts resolve to `student`).
- `backend/src/services/authService.ts` — `verifyAndSync` calls `userRepository.ensureAccount(...)` then injects `role = await roleService.resolveRole(uid)` into the returned account. Comment explicitly: role "is resolved (never taken from the client) and never written back by this call."
- `backend/src/controllers/authController.ts` — `verify` returns `{ account }` (account includes `role`). `registerTrainer` calls `trainerService.ensureTrainerAccount(uid, {...})`.

**Register-trainer endpoint — YES.**

- `backend/src/routes/auth.routes.ts`: `POST /verify`, `POST /link-trainer`, `POST /register-trainer`, `DELETE /account`. All behind `authenticate`.
- `register-trainer` body validated by `registerTrainerSchema` (`trainerValidators.ts`) = `{ name }` only — deliberately **no role field, no code field**.
- `trainerService.ensureTrainerAccount` sets `users/{uid}.role='trainer'`, upserts `trainers/{uid}`, generates a code if none. Idempotent.

**How verify returns role.** `/api/auth/verify` → `authService.verifyAndSync` → `{ ...account, role }`. Frontend `AccountInfo.fromJson` reads `account['role']` (default `'student'`); `isTrainer => role == 'trainer'`.

**Can a client set its own role? NO (secure).** `authenticate` always derives `uid` from the verified Firebase token (`middleware/auth.ts`, `req.uid = decoded.uid`), never from body/query. Role is only ever resolved server-side or set by `ensureTrainerAccount` from the verified uid. No endpoint accepts a `role` field in a body. ✅ No client-set-role risk found.

---

## B. LOGIN SCREEN / "TRAINER LOGIN" BUTTON

- `frontend/lib/features/auth/presentation/login_screen.dart` — there IS a `Trainer Login` button (`OutlinedButton.icon`, label `'Trainer Login'`). Its `onPressed` is the **same `_submit`** as SIGN IN (plain Firebase `signInWithEmailAndPassword`). Explanatory comment: "Trainer Login reuses the exact same Firebase email/password sign-in... the backend resolves the account role on /api/auth/verify." Helper text: "Trainers sign in with the same email and password."
- **No client path sets/chooses role at login.** The button does not set any flag, does not call register-trainer, does not pass any role. Role still comes only from `/api/auth/verify`.

**Verdict vs requirement:** Already compliant. A normal user tapping "Trainer Login" simply logs in and is routed to the **user** shell (because backend role is `student`). The requirement says "if a Trainer Login button exists it may only allow an already role=trainer account, else show an error." The current behavior is MORE permissive (it logs anyone in via that button but routes by role). Implementation decision needed: either (a) accept current behavior (role-correct routing already prevents privilege escalation), or (b) add an explicit guard that, when "Trainer Login" is used, errors if the resolved role != trainer. Current code does NOT error. **Low-risk gap; recommend (a) + optionally a post-verify check that shows an error if a non-trainer used the trainer entry point.**

---

## C. REGISTRATION ROLE SELECTION

- `frontend/lib/features/auth/presentation/register_screen.dart` — ALREADY has `_AccountType { user, trainer }` and an `_AccountTypeSelector` with two cards (`Key('accountType_user')`, `Key('accountType_trainer')`).
- `_submit()`: registers via `authControllerProvider`, then:
  - TRAINER → `registerTrainerControllerProvider.notifier.register(name)` (fire-and-forget; role set server-side).
  - USER → if the (optional) referral field is non-empty, `linkTrainerControllerProvider.notifier.link(code)`.
- The referral `TextFormField` (label "Trainer referral code") is shown **only for USER** accounts.
- Controllers live on the root container (`auth_controller.dart`: `RegisterTrainerController`, `LinkTrainerController`) so the call survives the screen being torn down by the auth redirect.

**What exists:** selector + trainer promotion + optional code entry. **What's missing for the new spec:** (1) the optional code field currently **links immediately** (auto-link) at registration — under the new approval model this should create a **pending request** instead (or be reframed); (2) labels say "referral" not "Trainer Code". See §E/§D.

---

## D. TRAINER CODE (rename + generation)

**Backend field names (keep stable).** Firestore uses `referralCode`, `referralCodeLower`, `referralCodeUpdatedAt` on `trainers/*`; the index collection is `referralCodes/{codeLower}`; link body field is `referralCode`. Endpoints: `GET/POST/PATCH /api/trainer/referral-code` and `GET /api/trainer/referral-code/availability` (`trainer.routes.ts`, `trainerController.ts`). The brief says keep backend field names stable; the rename is **UI-string only** (plus optionally new display aliases).

**Generator prefixes vs TRN- requirement.** `backend/src/utils/referralCode.ts` — `generateCandidate` picks a prefix from `['FIT','TRN','GYM','COACH']` + a suffix (7–8 chars total). The spec example is `TRN-XXXXXX`. Current generator produces e.g. `FIT7K2P9`, `TRN8X3Q` — **no hyphen, and the prefix is random (not always TRN)**. If the spec requires the literal `TRN-` prefix shape, the generator must change (and existing codes/tests `referralCode.test.ts`, `trainerService.test.ts` which assert current forms would need updating). Decision for implementer: (a) keep existing codes stable + only change the DISPLAY ("My Trainer Code" + show whatever code exists), or (b) change the generator to always `TRN-` + 6 chars. **Recommend (a)** to avoid churn and keep grandfathered codes (`dreamphysics`, `GYMRAHUL45`, etc.) valid; treat `TRN-XXXXXX` as an illustrative placeholder.

**Generation/persistence/uniqueness — DONE.** `trainerService.generateReferralCode` (retry loop + atomic `runClaimTransaction` against `referralCodes/{codeLower}`), `setCustomReferralCode`, `checkAvailability`; `trainerRepository.claimReferralCode/deactivateReferralCode/setActiveReferralCodeOnTrainer` enforce the shared claimability predicate transactionally. Codes are backend-generated only; uniqueness decided by the Firestore transaction. ✅

**UI strings that say "Referral Code"/"referral" and should become "Trainer Code"** (grep `[Rr]eferral` in `frontend/lib`):

- `frontend/lib/features/trainer/presentation/trainer_profile_screen.dart`:
  - header `'YOUR REFERRAL CODE'` (~line 132) → "MY TRAINER CODE".
  - snackbar `'Referral code copied'` (~line 159).
  - button labels `'Change / Generate new'` / `'Create a code'` and sheet title `'Change referral code'`, subtitle "Students who register with this code..." The spec wants `[Copy]` + `[Generate Trainer Code]` when none.
- `frontend/lib/features/profile/presentation/profile_screen.dart`:
  - `'Invalid trainer referral code'` snackbar (~line 833), `'Enter a trainer referral code to connect.'` (~line 899), `'Referral Code: ${trainer.referralCode}'` (~line 957), `'Enter the referral code your trainer gave you.'` (~line 1078), field labels `'Trainer referral code'` (~line 1090), field `Key('connectReferralField')`.
- `frontend/lib/features/trainer/presentation/trainer_students_screen.dart`: empty-state "Students who sign up with your referral code appear here." (~line 50) — under approval model this copy is also semantically wrong (they don't auto-appear).
- `frontend/lib/features/auth/presentation/register_screen.dart`: label/hint `'Trainer referral code'` / `'Enter trainer referral code (optional)'` (~lines 158–159).
- `frontend/lib/features/auth/presentation/verify_email_screen.dart`: `'Invalid trainer referral code'`, `"Couldn't verify the referral code. You can add it later."`.
- Dart identifiers (`referralCode`, `trainerReferralCodeProvider`, `ReferralCodeInfo`, widget keys) can stay as code names; only user-visible strings must change. Changing widget keys (`connectReferralField`, `customReferralField`) would break tests `connect_trainer_test.dart`, `trainer_referral_manage_test.dart` — change tests in lockstep if keys change.

**Trainer profile card shape vs spec.** Current `_ReferralCard` already shows the code + a Copy icon + a Change/Generate affordance. Spec wants: `My Trainer Code / TRN-XXXXXX / [Copy]`, and `[Generate Trainer Code]` when there is none. Current "No code yet" + "Create a code" path covers the no-code case; mostly a relabel.

**Student profile shows connected code.** `profile_screen.dart` `_MyTrainerSection` already renders `'Referral Code: ${trainer.referralCode}'` when linked (read-only text). Need: relabel to "Trainer Code", and ensure it is clearly read-only for an approved student (it already is — plain text). When unlinked, it shows "Connect to a trainer" (spec calls this "Add Trainer").

---

## E. LINK vs APPROVAL MODEL  ⚠️ THE CORE CHANGE

**Current model = immediate active link (NO pending/approval).**

- `backend/src/services/trainerService.ts` `linkStudent(studentUid, referralCode, {confirmSwitch})`:
  - resolves code via `referralCodes/{codeLower}` (`getByReferralCodeDoc`), verifies owner is an active trainer;
  - unknown/invalid → soft `{linked:false, reason:'invalid_code'}` (never errors);
  - trainer-as-student → 409;
  - already active to same trainer → `{linked:true, alreadyLinked:true}`;
  - active to a different trainer → `{linked:false, reason:'already_linked'}` unless `confirmSwitch:true` → `switchStudentTrainer`;
  - else `createStudentLink` → **transactionally writes `trainerLinks/{studentUid}.status='active'`**, mirrors `users/{uid}/profile/data.trainerId`, and `+1 totalStudents`.
- Status values seen in code/tests: **`'active'` and `'inactive'` only.** There is **NO `'pending'`** anywhere for trainer links (grep confirms `pending` appears only in `routineValidators`/routineService, unrelated). `trainerLinkRepository.listByTrainer` and `trainerRepository.countActiveStudents` both filter `status == 'active'`.
- `assertTrainerOwnsStudent` (`middleware/role.ts`) requires `link.status === 'active'`.
- Entry points to `linkStudent`: `POST /api/auth/link-trainer` (register-time) and `POST /api/student/connect-trainer` (`studentController.connectTrainer`, a thin alias) — both call the SAME `linkStudent`.

**What must change to support request → approve/reject:**

1. **Status enum / schema.** Introduce `'pending'` (and `'approved'`/`'rejected'` or reuse `'active'` for approved). Decision: keep `'active'` = approved to minimize churn to `listByTrainer`/`countActiveStudents`/`assertTrainerOwnsStudent`; add `'pending'` and `'rejected'`. Document in a status enum (no central enum exists today — add one, e.g. in `domain.ts`).
2. **Request creation.** New service method (e.g. `requestTrainer(studentUid, code)`) that validates the trainer/code exactly like `linkStudent` but writes `trainerLinks/{studentUid}` (or a new `trainerRequests` subcollection — see note) with `status:'pending'`, `trainerId`, timestamps, and does **NOT** mirror `profile.trainerId` and does **NOT** increment counts. Reject invalid code; reject duplicate pending/active request to the same trainer.
   - **Schema key collision risk:** today `trainerLinks` is keyed by `studentUid` (one doc per student = at most one trainer). A pending request to trainer B while active with trainer A cannot coexist under the same `trainerLinks/{studentUid}` doc. Options: (a) a single doc with `status` transitions (simplest, matches current switch logic, but a student can have only one in-flight relationship at a time — acceptable per spec "one trainer"); or (b) a separate `trainerRequests` collection keyed by `{trainerId}_{studentUid}`. **Recommend (a):** reuse `trainerLinks/{studentUid}` with `status` in `{pending, active(=approved), rejected, inactive}`. This keeps the single-trainer invariant and the existing ownership/count queries largely intact.
3. **Approve/Reject endpoints (trainer-only).** New `POST /api/trainer/requests/:studentUid/approve` and `.../reject` on `trainer.routes.ts` (behind `authenticate`+`requireTrainer`), with a NEW ownership check that the pending request's `trainerId == caller` (the existing `assertTrainerOwnsStudent` requires `status==='active'`, so it will NOT match a pending request — needs a `pending`-aware variant). Approve: set `status='active'` (=approved), mirror `profile.trainerId`, `+1 totalStudents` (reuse `createStudentLink`-style tx); Reject: set `status='rejected'` (or delete), no mirror/no count.
4. **Requests list query (trainer).** New method `listRequestsByTrainer(trainerId)` filtering `status=='pending'`, newest first. **Requires a new composite index** (see below).
5. **Students list query.** `listByTrainer` already filters `status=='active'` — approved students only. No change needed, but the Students tab should show **name + avatar only** initially (currently `TrainerStudentSummary` carries goal/weight/lastWorkout/todayActivity — the summary card can simply render only name+avatar; backend can keep returning the richer summary or be slimmed).
6. **student.trainerStatus field.** Spec says on approve set `student.trainerStatus = approved`. Today the student profile only mirrors `trainerId`. Add a `trainerStatus` mirror on `users/{uid}/profile/data` (and surface it on the student "My Trainer" card as read-only).
7. **Register-time + Profile "Add Trainer" behavior.** Change `register_screen` + `profile_screen` connect flow from "link immediately" to "send request" (call the new request endpoint; show "Request sent, waiting for approval").

**Student list query + composite index.**

- Query: `backend/src/repositories/trainerLinkRepository.ts` `listByTrainer` = `where('trainerId','==',t).where('status','==','active')` then in-memory sort by `updatedAt`.
- Index present: `firebase/firestore.indexes.json` has `trainerLinks` composite `(trainerId ASC, status ASC, updatedAt DESC)`. This index **already covers a `status=='pending'` ordered query too** (same fields), so a requests query ordered by `updatedAt`/`createdAt DESC` can reuse it **if** it orders by `updatedAt`. If requests are ordered by a different field (e.g. `createdAt`), add a new composite `(trainerId ASC, status ASC, createdAt DESC)`. NOTE: these queries run via the **Admin SDK** (backend), which does not strictly require a deployed composite index the way client SDK does, and `listByTrainer` already sorts in memory — so a new index may be optional, but add it for parity/consistency.

---

## F. MY STUDENTS UI (frontend)

**Screens that exist** (`frontend/lib/features/trainer/presentation/`):

- `trainer_shell.dart` — dedicated trainer bottom nav: **Dashboard / Students / Progress / Profile** (`trainerTabProvider`). NOT the student `HomeShell`.
- `trainer_dashboard_screen.dart`, `trainer_students_screen.dart`, `trainer_progress_screen.dart`, `trainer_profile_screen.dart`, `student_detail_screen.dart`, `widgets/trainer_widgets.dart`.
- Providers: `trainer_providers.dart`; repo: `trainer_repository.dart`; models: `trainer_models.dart`.

**Wired to REAL backend data (no mock).** `student_detail_screen.dart` is a 7-tab view (Overview, Workout, History, Nutrition, Water, Progress, Photos). Each tab watches a `FutureProvider.family` in `trainer_providers.dart` → `TrainerRepository` → the `/api/trainer/students/:uid/*` endpoints. Models in `trainer_models.dart` parse real fields and the header comment states "No field is fabricated." ✅ Requirement 4 "real data, no mock" is satisfied for the detail view.

**Gaps vs Requirement 4:**

- **No "Requests" tab/sub-view exists.** `trainer_students_screen.dart` is a single searchable list of active students only; there is no Students/Requests tab split and no Approve/Reject UI. Must ADD a tabbed Students|Requests screen (or add Requests to the trainer shell). No frontend provider/repo method for listing requests or approve/reject exists yet.
- **Students tab shows more than name+avatar.** `StudentSummaryCard` (in `trainer_widgets.dart`) renders goal/weight/activity. Spec wants name + avatar only initially → render a slim card; keep detail on tap.
- Tap already opens `StudentDetailScreen` with real data. ✅

**Backend endpoints the trainer uses to read a student (all re-verify ownership).** `trainer.routes.ts` (all behind `authenticate` + `requireTrainer`):

- `GET /api/trainer/profile`, `GET /api/trainer/students`
- `GET /api/trainer/students/:studentUid/overview|workouts|workouts/history|nutrition|water|progress|photos`
- Every per-student handler in `trainerController.ts` calls `await assertTrainerOwnsStudent(trainerId, studentUid)` BEFORE reading (and `listStudents` is scoped to the caller's own links). `assertTrainerOwnsStudent` throws 404 (not 403) on any mismatch to prevent enumeration. ✅ Backend authorization is real, not UI-only.

---

## G. ROLE-BASED NAV / ROUTING

- `frontend/lib/core/routing/app_router.dart` — `appRouterProvider` (go_router). Routes include `Routes.trainerHome` → `TrainerShell`, `Routes.home` → `HomeShell`.
- Redirect order: not-configured → (hold on splash until `authReadyProvider` settles) → (hold if `authStateProvider` loading) → signed-out → login/register/forgot allowed → not email-verified → verify-email → (hold if `accountInfoProvider` loading) → account error → startup-error → **role branch**: `if (account?.role == 'trainer') → Routes.trainerHome` (placed BEFORE the onboarding gate so a trainer skips onboarding) → else onboarding gate → home.
- `main.dart`/`app.dart` wire the router; `firebaseReadyProvider` gates not-configured.
- **My Students hidden for non-trainers:** YES by construction — the entire trainer surface lives in `TrainerShell`, reachable only when backend role == trainer. The student `HomeShell` (Home/Workout/Nutrition/AI/Profile) has no trainer-management entries. ✅

---

## H. SESSION / SHAREDPREFS / SPLASH  ⚠️ (requirement 6 largely MISSING)

- **SharedPreferences is initialized** in `frontend/lib/main.dart` (`SharedPreferences.getInstance()` → `sharedPreferencesProvider` override). But it is used only for: launch motivational message, theme, notifications, morning-game session start time. **There is NO storage of `firebaseUid`, `email`, or `role`.** (grep for `SharedPreferences`/`session`/`secureStorage` finds no auth-session keys.)
- **There is no secure storage** (`FlutterSecureStorage` not present). No token/password is stored anywhere (good — Firebase manages its own token persistence natively).
- **Current startup sequence (actual):**
  1. `main()` → init SharedPreferences + `FirebaseInitializer.initialize()`.
  2. Router gates on `firebaseReadyProvider`, then `authReadyProvider` (`authStateReady()` = awaits first settled `authStateChanges` event; see `auth_repository.dart` — custom `_settledInitialUser` works around the pinned `firebase_auth 5.7.0` lacking `authStateReady()` and the Android AOT leading-null race).
  3. `authStateProvider` (stream) resolves user; if signed-in + verified, `accountInfoProvider` calls `/api/auth/verify` (20s timeout → error state, not infinite spinner).
  4. Router reads `account.role` and routes trainer→`trainerHome`, else onboarding/home.
- **Role is already verified against the backend on startup** (`accountInfoProvider` → `verifyWithBackend` → `AccountInfo.role`). The router uses THAT, never a local hint. So the "backend/Firestore authoritative role verify" step already exists and is already the source of truth. ✅

**What's missing vs Requirement 6:** The literal "save `firebaseUid/email/role` to SharedPreferences on register/login; on reopen read local session metadata as a startup HINT" is **not implemented**. The app functionally meets the INTENT (correct role dashboard on reopen, no unnecessary login, backend is source of truth) WITHOUT a local hint. Implementation decision:

- **Option A (recommended, minimal risk):** Add a thin `SessionStore` writing `{firebaseUid,email,role}` to SharedPreferences on successful verify/login and clearing on logout, used ONLY as an optional display/analytics hint. Must NOT change the existing redirect logic (which must keep using the Firebase session + backend verify as the gates), to avoid regressing the carefully-fixed reopen race (see §task-fix-auth-reopen-and-fcm-churn). Where it slots: write in `accountInfoProvider`'s success path (or a listener on it) and in `authController.login`; clear in `logout`.
- **Option B:** Use the local role hint to pick an initial shell before verify completes — **NOT recommended**: risks showing the wrong shell briefly and re-introduces the exact "local overrides backend" anti-pattern the spec forbids.

Do NOT store password or token (spec + current design both forbid; honor it).

**No-write-on-reopen invariant (must preserve):** `userRepository.ensureAccount` deliberately writes nothing on an unchanged reopen; `addFcmToken` is a no-op when the token is unchanged. Any new session code must be client-local (SharedPreferences) and must not trigger extra Firestore writes on reopen.

---

## I. LOGOUT

- Student path: `profile_screen.dart` `_confirmLogout` → `authRepositoryProvider.logout()` → `_auth.signOut()`. On success the auth stream drives the router to Login.
- Trainer path: `trainer_profile_screen.dart` logout IconButton → `authControllerProvider.notifier.logout()` → `_repo.logout()` + resets `AuthActionState.idle`.
- `AuthController.logout` also resets its own state.

**Leakage risk analysis (Requirement 7):**

- Firebase `signOut()` clears the Firebase user; `authStateProvider` emits null; `currentUserProvider` → null.
- `accountInfoProvider` **watches `currentUserProvider`** and returns null when signed out, then re-fetches for the next user — so `role` cannot leak across accounts via that provider. ✅
- **RISK — stale trainer providers.** `trainerProfileProvider`, `trainerStudentsProvider`, `student*Provider` families, `myTrainerProvider`, `trainerReferralCodeProvider` are **plain `FutureProvider`s that do NOT watch `currentUserProvider`**. They are kept alive while the trainer shell is mounted; after logout the shell unmounts so they are disposed, but if NOT explicitly invalidated there is a theoretical window where cached trainer data (students, Trainer Code) could be read by a subsequently mounted widget before refetch. The API calls themselves carry the NEW user's token (so a refetch is correct), but **cached values are per-provider-container and are not auto-cleared on user switch.**
  - **Recommendation:** on logout, invalidate the auth/trainer provider tree (or scope these providers to the signed-in uid, e.g. `.family<_, uid>` or `ref.watch(currentUserProvider)` so they auto-dispose when the user changes). This directly addresses "Trainer A ... Trainer Code ... must not leak into User B." Add a widget/provider test (see §K `trainer_routing_test`).
- `homeTabProvider`/`trainerTabProvider` are `StateProvider<int>` — harmless UI index, but reset them for cleanliness.

---

## J. EXISTING-ACCOUNT ROLE DEFAULT

- `roleService.resolveRole`: a legacy `users/{uid}` doc with no `role` field and no `trainers/{uid}` doc resolves to `'student'`. It **never writes** the role back, so a legacy account is never auto-promoted. ✅
- `userRepository.ensureAccount` creates new accounts with **no `role` field** → they resolve to `student` until explicitly promoted by `ensureTrainerAccount`.
- Only `trainerService.ensureTrainerAccount` sets `role='trainer'`, and only for the verified uid that explicitly called `register-trainer`. No path auto-promotes an existing user. ✅
- Covered by `backend/src/__tests__/roleService.test.ts` (resolution order) — keep green.

---

## K. TESTS (what to extend / keep green)

**Backend (`backend/src/__tests__/`, run `npm test` / `npm run typecheck` / `npm run lint`):**

- `roleService.test.ts` — role resolution precedence (extend if adding `pending`/`approved` semantics don't affect role).
- `trainerService.test.ts` — in-memory Firestore mock; covers `linkStudent` create/switch/already-linked, `ensureTrainerAccount`, code generate/custom/availability, `assertTrainerOwnsStudent`, per-student reads. **Must extend** for request→approve/reject and `pending` status; existing immediate-link assertions (`status:'active'` on connect) will CHANGE if register/connect becomes request-based.
- `trainerRoutes.test.ts` — supertest-style; covers `/api/trainer/*` incl. ownership 404, PATCH custom code 409. **Extend** with approve/reject routes + requests list + pending-ownership.
- `referralCode.test.ts` — generator format (prefixes FIT/TRN/GYM/COACH, length). **Changes only if** the TRN- generator shape changes.
- `userRepository.test.ts` — `ensureAccount` no-write-on-reopen, `getRole`. Keep green (session/reopen invariant).
- `middleware.test.ts` — auth/validate/role middleware.
- Others (ai*, nutrition*, workout*, routine*, fitnessCalc, prCalc, streakCalc, health, batch2Validators, dailyPlan, planModificationDetection) — unrelated; keep green.

**Frontend (`frontend/test/`, run `flutter analyze` / `flutter test`):**

- `account_type_selector_test.dart` — register User/Trainer selector.
- `register_referral_test.dart` — register-time code entry (will change under approval model).
- `connect_trainer_test.dart` — Profile "connect to a trainer" sheet (keys `connectReferralField`); will change to "send request" semantics + label renames.
- `trainer_referral_manage_test.dart` — Trainer code manage sheet (key `customReferralField`); affected by label renames.
- `trainer_dashboard_test.dart`, `student_detail_test.dart`, `trainer_routing_test.dart` — trainer shell/detail/routing-by-role. **Extend** `trainer_routing_test` for logout-no-leak; add a Requests-tab test.
- `auth_restore_test.dart`, `auth_repository_test.dart`, `auth_error_mapper_test.dart` — startup/session restore; **guard the reopen invariant** when adding SharedPreferences session.
- `user_profile_test.dart`, `widget_test.dart`, plus unrelated (alarm/game/food/exercise/fcm/nutrition) — keep green.

---

## L. FIRESTORE RULES

`firebase/firestore.rules` (rules_version 2):

- `users/{uid}` + all subcollections: owner-only read/write (`isOwner(uid)`), with the top-level doc guarded so a client cannot set a mismatched `uid`.
- `trainers/{trainerId}`, `referralCodes/{code}`, `trainerLinks/{studentUid}`: **`allow read, write: if false`** — fully fail-closed; only the Admin SDK (backend) touches them. Default-deny trailer `match /{document=**} { if false }`.
- **Relevant to the approval model:** because `trainerLinks` is server-only, adding a `pending` status, a `trainerStatus` mirror on `users/{uid}/profile/data`, approve/reject writes, and (if chosen) a `trainerRequests` collection all stay **backend-only** and need NO client rule changes. If a new top-level collection (`trainerRequests`) is introduced, ADD an explicit `allow read, write: if false` match for self-documentation (mirrors the existing pattern; redundant with default-deny). The student `trainerStatus` mirror lives under `users/{uid}/profile/data`, already owner-readable — fine for the student to read their own status.

---

## GAP LIST (per the 8 end-state requirements)

1. **Roles (create-time select + persisted + login can't promote):** **DONE.** Files already correct: `register_screen.dart`, `authController.registerTrainer`, `roleService.ts`, `authService.ts`, `login_screen.dart` (Trainer Login is harmless). Optional tweak: explicit "non-trainer used Trainer Login" error.
2. **Trainer Code (rename + card shapes + student read-only + Add Trainer):** **PARTIAL.** Generation/uniqueness/persistence DONE. Must change: UI strings "Referral Code"→"Trainer Code" (`trainer_profile_screen.dart`, `profile_screen.dart`, `register_screen.dart`, `verify_email_screen.dart`, `trainer_students_screen.dart` copy); card layout to spec (`My Trainer Code / TRN-… / [Copy]` or `[Generate Trainer Code]`); confirm student code is read-only (already plain text). TRN- prefix shape decision (recommend display-only).
3. **Connection + Approval (pending → approve/reject):** **MISSING (core work).** Backend: `trainerService.ts` (new request/approve/reject methods; stop immediate link), `trainerLinkRepository.ts` (pending/requests queries), `middleware/role.ts` (pending-aware ownership check), `trainer.routes.ts`+`trainerController.ts`+`student.routes.ts`+`studentController.ts` (new endpoints; change connect to request), `trainerValidators.ts` (status/body), `domain.ts` (status enum), `firestore.indexes.json` (optional requests index), student profile mirror `trainerStatus`. Frontend: new Requests UI + "send request" UX in `profile_screen.dart`/`register_screen.dart`, new repo/provider methods.
4. **My Students (Students+Requests tabs, name+avatar only, real detail):** **PARTIAL.** Students list + real detail DONE. MISSING: Requests tab + Approve/Reject UI + backend requests endpoints; slim Students card to name+avatar. Files: `trainer_students_screen.dart`, `trainer_widgets.dart`, `trainer_repository.dart`, `trainer_providers.dart`, `trainer_models.dart`, trainer shell (add Requests).
5. **Role-based UI (user unchanged, student+trainer info, trainer-only own students, backend authz):** **DONE.** `app_router.dart` role branch, `TrainerShell` vs `HomeShell`, `assertTrainerOwnsStudent` backend authz. Student "My Trainer" card exists; just relabel + add `trainerStatus`.
6. **SharedPreferences + startup (save uid/email/role; reopen hint; backend authoritative):** **PARTIAL / mostly MISSING-literal.** Backend-authoritative startup + correct role routing on reopen already works (`app_router.dart`, `account_provider.dart`, `auth_repository.dart`). MISSING: the literal local `firebaseUid/email/role` persistence. Add a client-local `SessionStore` (new file under `frontend/lib/core/...`), write on login/verify-success, clear on logout; do NOT let it drive redirects. Preserve the no-write-on-reopen invariant.
7. **Logout / account switching (clear uid/email/role; no leak):** **PARTIAL.** Firebase sign-out + role re-fetch work. MISSING: explicit clearing of the (new) local session keys + invalidation/scoping of trainer providers to prevent cached Trainer A data lingering. Files: `auth_controller.logout`, `auth_repository.logout`, trainer providers (scope to uid or invalidate on logout), new `SessionStore.clear()`.
8. **Existing accounts (default user; never auto-promote; don't break features):** **DONE.** `roleService.resolveRole` + `userRepository.ensureAccount`. Regression surface = everything else; keep all existing tests green.

---

## RISK / REGRESSION LIST (most likely breakages)

- **Routing redirect race (HIGH sensitivity).** The reopen flow was explicitly hardened (`authReadyProvider`, `_settledInitialUser`, `_ProviderRefreshNotifier` listening to the exact providers the redirect reads). Do NOT make routing depend on a local role hint or add new async gates to the redirect; keep backend verify as the role source. Re-run `auth_restore_test.dart`.
- **No-write-on-reopen invariant.** `userRepository.ensureAccount`/`addFcmToken` avoid writes on unchanged reopen (see task-fix-auth-reopen-and-fcm-churn). New session persistence must be client-local only; don't add Firestore writes on reopen. Keep `userRepository.test.ts` green.
- **Provider leakage on logout (HIGH for req 7).** Trainer providers don't watch the current user; cached students/Trainer Code could persist across a user switch. Scope them to uid or invalidate on logout; add a test.
- **Student-data read authorization under the new pending status.** `assertTrainerOwnsStudent` requires `status==='active'`. If approval reuses `'active'` for approved, existing ownership checks keep working; a pending link must NOT grant data access. The approve/reject endpoints need a SEPARATE pending-ownership check — do not loosen `assertTrainerOwnsStudent`.
- **Immediate-link → request behavioral change ripples.** `linkStudent` is shared by register-time and Profile connect; both must switch to "request." Existing tests asserting `status:'active'` after connect (`trainerService.test.ts`, `trainerRoutes.test.ts`, `connect_trainer_test.dart`, `register_referral_test.dart`) WILL need updating — update them in lockstep, not by weakening the new behavior.
- **Widget-key / label renames break tests.** `connectReferralField`, `customReferralField` keys and "Referral code" strings are asserted in tests. Rename tests and code together.
- **Switch-trainer UX vs single pending slot.** Reusing `trainerLinks/{studentUid}` for pending means a student can have only one in-flight relationship; confirm this matches the "one trainer" rule (it does) and define what happens to an existing active link when a new request is sent (recommend: block new request until current is removed, or auto-supersede on approve).
- **TRN- generator change (if chosen) churns existing codes + tests.** Prefer display-only; if changing the generator, migrate/grandfather existing codes and update `referralCode.test.ts`.

---

## RECOMMENDATIONS (not implemented here)

1. Treat Requirement 3 as the backbone feature: add `pending`/approve/reject on `trainerLinks/{studentUid}`, keep `'active'`=approved; add a pending-aware ownership guard; new trainer approve/reject + requests-list endpoints; switch register/connect to "send request."
2. Do UI renames ("Trainer Code") and the Students-tab slim-card + new Requests tab alongside, reusing existing providers/repo patterns.
3. Add a client-local `SessionStore` for `firebaseUid/email/role` as a hint only; clear on logout; never override backend role; preserve the reopen no-write invariant.
4. Scope/invalidate trainer providers on logout to guarantee no cross-account leakage.
5. Keep backend field names (`referralCode*`) stable; treat `TRN-XXXXXX` as illustrative (display-only) unless the user insists on the literal shape.
