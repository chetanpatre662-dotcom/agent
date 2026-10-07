# Design — FitTrack AI Trainer ⇄ Student 2-Way System (+ Nutrition Add-Food bug)

Status: revision pass (addresses `design-review.json`, verdict CHANGES_REQUESTED — 2 HIGH, 5 MEDIUM, 4 NIT). Responses to each finding are recorded in §12.
Scope: additive trainer layer on top of the existing student app, plus a root-cause fix for the Nutrition → Add-Food flow. Design only — no production code is written in this step.

Revision summary: the nutrition root cause is now identified as a concrete, static-provable backend read defect (§7), not a list of runtime candidates. The router role branch is pinned to fire **before** the onboarding gate with the exact role-wiring call sites named (§6.1, §5.1). The referral-link call site and invalid-code surfacing are pinned to the verify-email screen (§6.4). Add-Food idempotency is resolved to a single client-supplied-id approach with the exact schema/service/model changes (§7). Signed-URL credential constraint, non-active link handling, `totalStudents` live-count, the single composite index, and the no-write role resolution are all specified.

Worktree note: the task names a worktree at `.worktrees/trainer-student-2way/{backend,frontend,firebase}`. That worktree did not exist at design time, so investigation was performed against the live repository it branches from:
- Backend: `c:\Users\cheta\OneDrive\Desktop\fit\backend` (Node + TypeScript + Express, vitest)
- Flutter: `c:\Users\cheta\OneDrive\Desktop\fit\frontend` (Riverpod + go_router + Dio + Firebase)
- Firestore rules: `c:\Users\cheta\OneDrive\Desktop\fit\firebase\firestore.rules`; Storage rules: `firebase\storage.rules`
Paths below are given relative to those roots; the implementer applies them inside the worktree if/when it is created, otherwise in-repo. The structure is identical.

---

## 1. Overview

The app is a layered Express backend (`routes → controllers → services → repositories`, Zod `validators`, shared `middleware/auth.ts` that verifies Firebase ID tokens and sets `req.uid`) over Firestore, with a Flutter client (feature folders of `data / presentation / providers`, Riverpod providers, a `go_router` with auth-aware redirects, and a Dio `ApiClient` that attaches the Firebase ID token and unwraps a `{ success, data }` / `{ success, error }` envelope). All private data is namespaced under `users/{uid}/...` and locked to the owner by Firestore rules; reference data (`exercises`, `foods`, …) is read-only to clients. Routing after sign-in is already driven by backend-returned account metadata from `POST /api/auth/verify` (`AccountInfo` → `onboardingCompleted` chooses onboarding vs. home).

The trainer system is **purely additive**. We extend the existing `users/{uid}` account document with a server-trusted `role` field, add a `trainers/{trainerId}` collection and a `trainerLinks/{studentUid}` link collection, seed one trainer (`dreamphysics` / "Dream Physics"), add a new set of **trainer-only** backend endpoints that re-verify ownership on every call, add a role-based branch to the Flutter router (trainer → new Trainer shell; student → existing `HomeShell` untouched), and tighten Firestore rules fail-closed. No existing student collection, screen, route, provider, or the Midnight Energy bottom-nav shell is modified in a breaking way. Trainer reads **reuse** the student's existing subcollections (`workouts`, `foodLogs`, `waterLogs`, `bodyMeasurements`, `personalRecords`, `progressPhotos`, `profile/data`) via the Admin SDK — no data is duplicated.

The chosen stack is locked: **TypeScript/Express/Firebase Admin/Zod/vitest** on the backend, **Flutter/Riverpod/go_router/Dio/firebase_auth** on the client, **Firestore + Firebase Storage** for data. No new frameworks, databases, or auth systems are introduced.

---

## 2. Open decisions (resolved)

### 2.1 Invalid referral code during registration → create an **unlinked student**, surface a non-blocking warning
Registration is a Firebase Auth operation that happens entirely client-side (`createUserWithEmailAndPassword`), and the referral link is a **separate, later** server step. The least-surprising, least-destructive behavior is: **always create the student account; attempt to link; if the code is unknown, create a normal independent student and return a soft, non-fatal signal** so the UI can show `Invalid trainer referral code — you've been set up as a regular member. You can add a trainer later.`

Reasoning: blocking registration would mean either (a) deleting an already-created Firebase Auth user (fragile, racy, and can strand a half-created account) or (b) blocking before auth creation, which can't validate the code without an authenticated call. A typo in an *optional* field must never cost someone their account. The requirement "Empty → normal independent student" and "Invalid → show 'Invalid trainer referral code'" are both honored: the account is created as a student, and the client shows the invalid-code message. The exact client copy is specified in Acceptance Criteria.

Edge case — valid-but-wrong-casing: referral codes are normalized to lowercase/trimmed on both input and storage, so `DreamPhysics` links the same as `dreamphysics`.

### 2.2 Trainer ⇄ student link shape → **dedicated `trainerLinks/{studentUid}` top-level collection + mirrored `trainerId` on the student profile**
Two complementary pieces, both keyed by **IDs** (never email):

- `trainerLinks/{studentUid}` (top-level): the authoritative relationship document `{ studentUid, trainerId, status, createdAt, updatedAt }`. Keyed by `studentUid` so a student has at most one trainer and the link is a single-doc lookup. Queried by `where('trainerId','==', X)` to list a trainer's students — this is the trainer's student list.
- `users/{studentUid}/profile/data.trainerId` (mirror): a denormalized convenience field for the student's own "My Trainer" screen and for rule checks, written transactionally with the link.

Reasoning: the existing schema keeps everything either under `users/{uid}` (owner-only) or as top-level reference data. A trainer must query *across* students, which owner-scoped subcollections can't express efficiently, so a **top-level link collection** is the right shape — it is the only cross-user index in the system and is never client-writable (backend Admin SDK only). We avoid a `trainers/{trainerId}/students/{studentUid}` subcollection as the *source of truth* because a per-student single-doc lookup (`trainerLinks/{studentUid}`) is cheaper and makes "who is this student's trainer?" O(1). We do **not** duplicate student profiles — the link holds only IDs + status; all student data is read live from `users/{studentUid}/...`.

`status`: `active | inactive | pending | removed` (string enum). Current flow only produces `active`.

### 2.3 Progress-photo visibility → **explicit per-student opt-in flag `shareProgressWithTrainer` on the student profile; trainer reads photos via a backend-signed URL, never direct Storage**
Progress photos store metadata in `users/{uid}/progressPhotos/{id}` and bytes in Storage under `users/{uid}/`, which Storage rules make **owner-only** — a trainer's Firebase token cannot read another user's Storage object directly, by design, and we keep it that way. The opt-in model:

- Add `profile/data.shareProgressWithTrainer: boolean` (default **false**). The student toggles it from "My Trainer".
- Trainer photo endpoint returns `403` (not `404`) when the flag is false, so the UI can say "This student hasn't shared progress photos."
- When the flag is true, the backend (Admin SDK, which bypasses Storage rules) generates a **short-lived V4 signed read URL** (TTL 15 min) per photo and returns it to the trainer. Photos are therefore never public and never require loosening Storage rules.

Reasoning: this reuses the existing "metadata in Firestore, bytes in Storage, owner-only" split and the existing `ProgressPhotoService.assertOwnedPath` invariant. Signed URLs are the standard way to grant scoped, expiring access without making a bucket path world-readable. A per-student boolean (vs. per-photo) matches the requirement wording ("photos the student made available for trainer monitoring") and keeps the student's control simple; per-photo granularity is noted as a future extension but out of scope now.

**Credential constraint (must be honored by the implementer).** V4 `getSignedUrl({ version:'v4', action:'read', expires })` requires a service-account **private key** to sign. `config/firebase.ts` supports both inline cert credentials (`privateKey` present → signing works) and a `GOOGLE_APPLICATION_CREDENTIALS` JSON file (also key-bearing → works). Under Application Default Credentials **without** a key (some CI/emulator setups), `getSignedUrl` throws. The design therefore: (a) states explicitly that the signed-URL path requires key-bearing credentials; (b) handles a per-photo signing failure gracefully (§5.4 returns `url:null` for that photo, never failing the whole list); and (c) requires that the emulator / rules-test slice must **not** assert real signed URLs — those are integration-only against key-bearing creds. The implementer must not wire a CI test that depends on real signed-URL generation.

### 2.4 Role seeding for the initial trainer → **idempotent seed script + `role` claim resolution at verify time**
- Add `backend/src/scripts/seedTrainer.ts` (mirrors existing `scripts/seedExercises.ts`), run via a new `package.json` script `seed:trainer`. It takes the trainer's Firebase UID (from an env var `SEED_TRAINER_UID`, obtained by creating the trainer's Firebase Auth user once via the console or CLI) and upserts:
  - `users/{trainerUid}.role = 'trainer'`
  - `trainers/{trainerId}` with `{ trainerId: trainerUid, name: 'Dream Physics', email, referralCode: 'dreamphysics', referralCodeLower: 'dreamphysics', totalStudents: 0, status: 'active', createdAt, photoUrl: null }`
  - `referralCodes/dreamphysics → { trainerId }` (lookup index; see §5.2)
  The script is idempotent (merge writes; safe to re-run).
- The seed is the *only* hardcoded trainer datum, satisfying "no hardcoded data except the trainer referral code."

Reasoning: Firebase Auth users are created through Firebase, not our DB; a seed script that promotes an existing UID to `trainer` and creates the trainer profile is the minimal, convention-matching way to bootstrap. We deliberately **do not** auto-promote anyone based on client input — role is set only by the seed (or future admin tooling).

### 2.5 Legacy accounts resolve to `student`
`role` is **optional** in storage. Resolution rule (in a new `roleService`, called from `authService.verifyAndSync`): `effectiveRole = users/{uid}.role ?? (trainers/{uid} exists ? 'trainer' : 'student')`. Any legacy account with no `role` field and no `trainers/{uid}` doc is a `student`. **`resolveRole` is purely computed on each verify and never writes the resolved value back** to `users/{uid}.role`; writing a resolved `student` would add a write on an otherwise-unchanged reopen and break the existing no-write-on-unchanged-reopen invariant. `ensureAccount` already only patches `email`/`emailVerified` and returns the existing doc otherwise, so an explicitly-set `role` survives untouched; no change to its write condition is needed beyond confirming it never clears `role`. `verify` returns the resolved role so the client never guesses.

---

## 3. Data model & Firestore schema changes

All new fields are additive; existing documents remain valid.

### 3.1 Extend `users/{uid}` (account doc)
Add optional `role?: 'student' | 'trainer'`. `UserRepository.ensureAccount` keeps its current conditional-write behavior and must **never** write or clear `role` implicitly. A new `UserRepository.getRole(uid)` reads it.

### 3.2 Extend `users/{uid}/profile/data`
Add optional:
- `trainerId?: string | null` — mirror of the authoritative link (denormalized).
- `shareProgressWithTrainer?: boolean` (default false).
Both added to `profileUpsertSchema` as optional so existing upserts stay valid; neither is required for onboarding.

### 3.3 New top-level `trainers/{trainerId}` (trainerId == trainer's Firebase UID)
```jsonc
{
  "trainerId": "string",      // == Firebase UID of the trainer account
  "name": "Dream Physics",
  "email": "string|null",
  "photoUrl": "string|null",
  "referralCode": "dreamphysics",      // display form
  "referralCodeLower": "dreamphysics", // normalized unique key (future: unique per trainer)
  "totalStudents": 0,          // CONVENIENCE counter maintained via transaction on link create/remove; the authoritative display value is a live count query on trainerLinks (see §5.3)
  "status": "active|inactive",
  "createdAt": "timestamp",
  "updatedAt": "timestamp"
}
```

### 3.4 New top-level `referralCodes/{codeLower}` (lookup index)
```jsonc
{ "trainerId": "string", "createdAt": "timestamp" }
```
Doc id = normalized code (e.g. `dreamphysics`). Guarantees codes are **structurally unique per trainer** (one doc per code) and makes code→trainer an O(1) lookup during registration. Future trainers get their own unique code doc.

### 3.5 New top-level `trainerLinks/{studentUid}` (authoritative relationship)
```jsonc
{
  "studentUid": "string",
  "trainerId": "string",
  "status": "active|inactive|pending|removed",
  "createdAt": "timestamp",
  "updatedAt": "timestamp"
}
```
Trainer's student list = `trainerLinks where trainerId == <me> and status == 'active'`. Requires a composite index.

### 3.6 `firestore.indexes.json`
Add exactly **one** composite index: collection `trainerLinks`, fields in order `trainerId ASC, status ASC, updatedAt DESC`. This single three-field index covers the equality-on-`trainerId` + equality-on-`status` + order-by-`updatedAt DESC` query used for the student list and recent-activity ordering (equality + equality + orderBy needs precisely this one composite — not two separate indexes). The existing `bodyMeasurements (type, measuredAt)` entry is the precedent for the JSON shape.

### 3.7 Writer ownership
`trainers`, `referralCodes`, `trainerLinks`, and the `trainerId`/`shareProgressWithTrainer` profile mirror are **only ever written by the backend Admin SDK** (seed script + link service inside a Firestore transaction). Clients never write them (enforced by rules, §8).

---

## 4. Firestore & Storage security rules

`firestore.rules` is updated fail-closed. The existing `users/{uid}` owner-only block and default-deny are preserved. Add:

```
// Relationship + trainer data: NEVER client-writable (backend Admin SDK only),
// and NOT client-readable here (served exclusively through authorized backend
// endpoints that re-check the token+role+ownership). Fail-closed.
match /trainers/{trainerId}     { allow read, write: if false; }
match /referralCodes/{code}     { allow read, write: if false; }
match /trainerLinks/{studentUid}{ allow read, write: if false; }
```

Rationale: trainer reads of *student* data must be mediated by the backend (which can enforce "trainer owns this student"); Firestore rules cannot express that cross-user predicate cleanly, so the client is denied direct access and must go through the authorized API. The Admin SDK bypasses rules, so the seed script and link service still work. This keeps **no public read of any student data** and prevents a trainer client from reading another trainer's links directly. Student data under `users/{uid}/**` stays owner-only exactly as today — a trainer never reads it with their own Firebase token.

`storage.rules`: **unchanged**. Trainers get progress-photo bytes only through backend-signed URLs, so no rule relaxation is needed (and we must not add one — it would risk public exposure).

New rule tests (emulator, in the existing rules-test slice): trainer cannot read `trainerLinks` directly; student cannot read another student's `users/{other}/**`; `trainers`/`referralCodes`/`trainerLinks` are not client-writable; unauthenticated denied everywhere.

---

## 5. Backend design

New feature slice following existing conventions exactly: `routes/trainer.routes.ts`, `controllers/trainerController.ts`, `services/trainerService.ts` (+ `roleService.ts`), `repositories/trainerRepository.ts` (+ `trainerLinkRepository.ts`), `validators/trainerValidators.ts`, tests under `src/__tests__/`.

### 5.1 Role resolution & routing data (extends existing auth)
- `roleService.resolveRole(uid)` → `users/{uid}.role ?? (trainers/{uid} exists ? 'trainer' : 'student')`. This resolver is **purely computed per verify and NEVER writes `role` back** to `users/{uid}` — persisting a resolved `student` would add a write on reopen and break the existing "no-write-on-unchanged-reopen" invariant that `ensureAccount` guarantees. The resolved value is only returned in the response, never stored.
- `authService.verifyAndSync` is the exact wiring point: it currently returns `userRepository.ensureAccount(...).account` verbatim (an object with no `role`). It is extended to call `roleService.resolveRole(uid)` and **inject** `role` into the returned account object before returning. `authController.verify` already returns that account via `ok(res, { account })`, so `POST /api/auth/verify` response `account` then carries `role` with no controller change.
- `AccountInfo.fromJson` (Flutter, `auth_repository.dart`) gains `final String role;` parsed as `account['role'] as String? ?? 'student'` (default `student` for any legacy/absent value).
- New middleware `requireTrainer` (new `middleware/role.ts`): after `authenticate`, calls `roleService.resolveRole(req.uid)` and throws `403 ForbiddenError` if not `trainer`. Reused by every trainer route.
- New helper `assertTrainerOwnsStudent(trainerId, studentUid)`: reads `trainerLinks/{studentUid}`; throws `404 NotFoundError` if the link is missing, OR `trainerId` mismatches, OR `status !== 'active'` (any of `inactive | pending | removed` yields 404 — a future deactivated/removed link can never be read). Returning `404` (not `403`) avoids leaking the existence of another trainer's student. Called at the top of **every** per-student endpoint.

### 5.2 Registration linking (extends auth)
New endpoint `POST /api/auth/link-trainer` (authenticated; body `{ referralCode: string }`), called by the client immediately after `register` + first `verify`:
- Normalize code (`trim().toLowerCase()`).
- Lookup `referralCodes/{code}`. If absent → return `{ linked: false, reason: 'invalid_code' }` with HTTP `200` (soft, non-blocking per §2.1). The account remains a plain student.
- If present, run a Firestore **transaction**: create `trainerLinks/{studentUid}` (if absent) with `status:'active'`, set `profile/data.trainerId`, increment `trainers/{trainerId}.totalStudents`. Idempotent: if a link already exists for this student, do not double-count. Return `{ linked: true, trainerId, trainerName }`.
- Guard: a `trainer`-role account cannot be linked as a student → `409 ConflictError`.

Why a transaction: the link doc, the profile mirror, and `totalStudents` must stay consistent; a transaction prevents double-increment on retries/duplicate taps.

### 5.3 Trainer endpoints (all behind `authenticate` + `requireTrainer`)
`routes/trainer.routes.ts` mounted at `/api/trainer` in `app.ts`.

| Method & path | Purpose | Reuses / returns |
|---|---|---|
| `GET /api/trainer/profile` | Trainer profile for Profile tab | `trainers/{me}` + `totalStudents` from a **live count query** on `trainerLinks where trainerId==me, status='active'` (authoritative for display). The stored `trainers/{me}.totalStudents` counter is a convenience only and may be shown as a fallback; the live count wins so a drifted counter (e.g. from a partially-failed transaction) never displays. |
| `GET /api/trainer/students` | Student list for Dashboard/Students | `trainerLinks where trainerId==me,status=active`, then per student a **summary** aggregated from `profile/data`, latest `workouts`, today's `foodLogs`/`waterLogs` |
| `GET /api/trainer/students/:studentUid/overview` | Detail → Overview tab | student `profile/data` (age/gender/height/weight/goal/activity/schedule) + association + progress summary |
| `GET /api/trainer/students/:studentUid/workouts?date=` | Detail → Workout tab | `users/{s}/workouts` (embedded exercises/sets: sets, reps, weight, duration, cardio, calories, status) |
| `GET /api/trainer/students/:studentUid/workouts/history?limit=` | Detail → Workout History | `users/{s}/workouts` list + `personalRecords`; volume/duration via existing `workoutCalc`/`prCalc` |
| `GET /api/trainer/students/:studentUid/nutrition?date=` | Detail → Nutrition (daily + historical) | `nutritionService.getDay(studentUid, date)` reused directly (calories/protein/carbs/fat/fiber/sugar/sodium, meals, items, timing) |
| `GET /api/trainer/students/:studentUid/water?date=` | Detail → Water | `users/{s}/waterLogs` + `profile.targets.waterMl` |
| `GET /api/trainer/students/:studentUid/progress` | Detail → Progress | `bodyMeasurements` (weight history, measurements), goal progress, workout/nutrition consistency computed from existing data |
| `GET /api/trainer/students/:studentUid/photos` | Detail → Progress Photos | gated by `shareProgressWithTrainer`; returns metadata + short-lived signed URLs |

Student-facing addition:
| `GET /api/student/trainer` | "My Trainer" card | `trainerLinks/{me}` → `trainers/{trainerId}` public-safe subset (name, photoUrl, email if shared, status). A link whose `status !== 'active'` (any of `inactive \| pending \| removed`) is treated as **no trainer** → returns `{ trainer: null }`, so a deactivated/removed link does not surface a trainer. |
| `PATCH /api/student/trainer/sharing` | Toggle photo sharing | sets `profile/data.shareProgressWithTrainer` |

Each per-student handler calls `assertTrainerOwnsStudent(req.uid, studentUid)` **first**. The services reuse existing per-uid services/repositories by passing `studentUid` as the uid argument — no new aggregation logic is duplicated where an existing service already computes it (e.g. nutrition day totals use `nutritionService.getDay`).

### 5.4 Error handling (concrete, per operation)
Uses existing `utils/errors.ts` (`AppError` subclasses) + `middleware/errorHandler.ts` (maps to `{ success:false, error:{ code, message } }`). The existing `ok()` envelope is used for success.

| Condition | Recoverable? | Caller receives | Logged? |
|---|---|---|---|
| Missing/invalid token (any trainer route) | yes (re-auth) | `401 UnauthorizedError` | `warn` (code only, no token) — existing behavior |
| Authenticated but role≠trainer | no | `403 ForbiddenError` "Trainer access required." | `warn` with uid |
| Trainer requests a student not linked to them (or another trainer's) | no | `404 NotFoundError` "Student not found." (deliberately not 403) | `warn` with trainerId+studentUid |
| Student doc/subcollection empty for a date | yes (expected) | `200` with empty arrays / zeroed totals (not an error) | none |
| Photo endpoint, sharing off | n/a | `403` "Student hasn't shared progress photos." | none |
| Referral code unknown (`link-trainer`) | yes | `200 { linked:false, reason:'invalid_code' }` | `info` |
| Referral code valid, link already exists | yes | `200 { linked:true }` idempotent | none |
| Trainer account tries to link as student | no | `409 ConflictError` | `warn` |
| Firestore/Admin unavailable | maybe | `503 ServiceUnavailableError` (existing pattern) | `error` |
| Signed-URL generation fails | partial | `200` with that photo's url `null` + others intact; never fail the whole list | `warn` |
| Unexpected error | no | `500` generic (errorHandler) | `error` with stack |

### 5.5 Input validation (Zod, per external input)
- `linkTrainerSchema`: `{ referralCode: z.string().trim().toLowerCase().min(3).max(40) }` — required; on failure `422` via `validate` middleware.
- `studentUidParamsSchema`: `{ studentUid: z.string().min(1).max(128) }` — required; `422` on failure.
- `dateQuerySchema`: reuse `commonSchemas.dateKey.optional()`; invalid date → `422`.
- `historyQuerySchema`: `{ limit: z.coerce.number().int().min(1).max(365).optional() }`.
- `sharingPatchSchema`: `{ shareProgressWithTrainer: z.boolean() }`.
All follow the existing `validate({ params, query, body })` convention from `nutrition.routes.ts`.

### 5.6 Invariant ownership
- "A trainer reads only their own students" — enforced in the **service/middleware layer** (`assertTrainerOwnsStudent`), because it is a cross-document predicate the Firestore rules can't express for client reads, and because all student data access is funneled through the backend. This is the single enforcement point for every per-student endpoint.
- "Role is server-trusted" — enforced at the **auth/verify + middleware layer**; the client value is never consulted.
- "Link consistency / totalStudents" — enforced at the **repository/transaction layer**.

---

## 6. Flutter design

New feature folder `lib/features/trainer/` with `data/`, `presentation/`, `providers/`, mirroring existing features. The student experience and `HomeShell` (Midnight Energy bottom nav) are **not modified** except for the additive items in §6.4–6.5.

### 6.1 Role in `AccountInfo` + routing
- `AccountInfo` (in `auth_repository.dart`) gains `final String role;` parsed from `account['role'] as String? ?? 'student'`.
- `app_router.dart` redirect — the role branch must be inserted **BEFORE the existing onboarding gate**, not after. The current redirect evaluates `if (!onboarded) return Routes.onboarding;` and then force-routes onboarded users to `Routes.home`; a branch placed *after* the onboarding gate would send a freshly seeded trainer (`onboardingCompleted:false`) into the **student** fitness onboarding and never reach the trainer branch. The correct insertion point is immediately after the account has loaded successfully (after the `accountAsync.isLoading` / `hasError` holds, before `final onboarded = ...`):

  ```dart
  final account = accountAsync.valueOrNull;

  // Role branch — BEFORE the student onboarding gate.
  // Trainers have no fitness onboarding, so this must precede it.
  if (account?.role == 'trainer') {
    const trainerLoc = Routes.trainerHome;
    final onTrainer = loc == trainerLoc; // (or startsWith for nested trainer routes)
    return onTrainer ? null : trainerLoc;
  }

  // --- existing student logic unchanged below ---
  final onboarded = account?.onboardingCompleted ?? false;
  if (!onboarded) { return loc == Routes.onboarding ? null : Routes.onboarding; }
  // ... existing "keep out of auth/onboarding/splash" block → Routes.home
  ```
  The redirect reads role only from the already-watched `accountInfoProvider` (trusted backend state) — never from local referral input. A `student`-role account hits none of the trainer branch and flows through the unchanged student logic verbatim.
- Add route constant `trainerHome` (`/trainer`) in `route_names.dart` and a `GoRoute(path: Routes.trainerHome, builder: (_, __) => const TrainerShell())`. The trainer tab routes are internal to a trainer shell (IndexedStack like `HomeShell`), so a single `/trainer` route is sufficient; deep sub-navigation (student detail) uses `Navigator.push` within the shell, matching how `HomeShell` pushes the morning-challenge screen.

### 6.2 Trainer shell & navigation (role-based, separate from student shell)
`lib/features/trainer/presentation/trainer_shell.dart` — its own `Scaffold` + bottom nav reusing the **same Midnight Energy visual components** (`_MidnightNavBar` pattern, `AppColors`, `AppShadows`) so the theme is consistent, but with trainer tabs: **Dashboard, Students, Progress, Profile**. This is a distinct widget tree; the student `HomeShell` is untouched (keeps its 5 student tabs). Role-based routing (not nav-jamming) per requirement 9.

### 6.3 Trainer screens (reusing existing analytics/chart widgets)
- `trainer_dashboard_screen.dart` — Overview cards (Total/Active students, Today's Activity, Workout Sessions, Students needing attention, Recent activity) + student list with **search / sort / filter** (All/Active/Inactive/Recently active/Goal/Progress) implemented client-side over the `GET /students` payload. Student card: photo, name, goal, current weight, progress summary, last workout, today's activity status, nutrition status, overall progress.
- `trainer_students_screen.dart` — fuller student list (same data source).
- `student_detail_screen.dart` — `TabBar`/`TabBarView` with tabs **Overview / Workout / Workout History / Nutrition / Water / Progress / Progress Photos**, each backed by a `FutureProvider.family` keyed by `studentUid` (+ date where relevant). Charts reuse existing progress/nutrition chart widgets (`MacroRing`, progress charts) by feeding them the trainer-fetched data — the widgets are presentation-only and uid-agnostic.
- `trainer_progress_screen.dart` — cross-student analytics aggregated from the students payload.
- `trainer_profile_screen.dart` — trainer profile + referral code display.

### 6.4 Student additions (non-breaking)
- `register_screen.dart`: add ONE optional `TextFormField` "Trainer referral code" with placeholder `Enter trainer referral code (optional)`, no validator that blocks submission. Empty field → skip the link call entirely (pure existing flow).

  **Call-site and message-surfacing decision (resolves the email-verify-redirect timing).** `register_screen._submit` → `authController.register` → `createUserWithEmailAndPassword` + `sendEmailVerification`. The user is now signed-in-but-unverified, so the router immediately redirects to `Routes.verifyEmail` and the register screen is torn down — a SnackBar on the register screen would never be seen. Therefore:
  1. When the referral field is non-empty, call `authRepository.linkTrainer(code)` **right after a successful `register()`**, while the fresh Firebase token is valid. `POST /api/auth/link-trainer` only requires `authenticate` (not email verification), so it works pre-verification.
  2. The link **result is persisted into a Riverpod state channel** (a `linkTrainerResultProvider` / `StateProvider<LinkTrainerResult?>`), not shown on the soon-to-be-popped register screen.
  3. The **verify-email screen** reads that provider and surfaces the outcome there: on `{linked:false, reason:'invalid_code'}` it shows the "Invalid trainer referral code" message (per §2.1 copy, non-blocking — the account still exists as a student); on `{linked:true}` it shows a confirmation ("Connected to Dream Physics"). The provider is cleared once shown. This guarantees the invalid-code message is always surfaced on a screen that is actually mounted after registration.
  4. `linkTrainer` is a distinct repository method hitting `/api/auth/link-trainer`; it is **not** folded into `register` or `verifyWithBackend`. It is called exactly once, immediately after register succeeds.
- `login_screen.dart`: add a clear secondary action **"Trainer Login"**. It uses the **same** Firebase email/password sign-in (no separate auth). Design: "Trainer Login" opens the same login form (or toggles a label) and signs in via the existing `authController.login`; role is resolved by the backend post-login and the router routes to the trainer shell. No separate credential path, no client-side role decision.
- "My Trainer" card: a new section in the existing student `profile_screen.dart` (additive widget) showing trainer name/photo/status from `GET /api/student/trainer`, plus the `shareProgressWithTrainer` toggle. Example copy: "Your Trainer — Dream Physics — Referral Code: dreamphysics".

### 6.5 Data layer
`lib/features/trainer/data/trainer_repository.dart` wraps `ApiClient` for all `/api/trainer/*` and `/api/student/trainer` calls, returning `Result<T>` + typed models (`TrainerProfile`, `TrainerStudentSummary`, `StudentOverview`, reuse existing `NutritionDay`, `FoodLogEntry`, workout/measurement models where shapes match). Providers in `lib/features/trainer/providers/` expose `FutureProvider`/`FutureProvider.family` matching the existing nutrition/profile provider style.

### 6.6 Edge cases (client)
- Trainer with zero students → empty-state view, not an error.
- Student with no data for a date → zeroed/empty tab content (backend returns `200`).
- Photo sharing off → tab shows the "not shared" message from the `403`.
- Signed URL expired mid-view → pull-to-refresh re-fetches (invalidate the family provider), mirroring nutrition's `RefreshIndicator`.
- A student who later has their link removed → trainer detail calls return `404`; UI shows "no longer linked" and pops.

---

## 7. Nutrition "Add Food" bug — root cause (identified) + fix

The task requires tracing the **complete** flow and fixing the actual root cause, not patching UI. The full chain was traced and the defect is **static-provable** — it does not require runtime guesswork.

Full flow traced:
`NutritionScreen._addFood` → `AddFoodSheet` → search (`foodSearchProvider` → `GET /api/nutrition/foods/search`) or custom form → `_buildEntry()` builds a `FoodLogEntry` → `nutritionRepository.addFood(entry)` → `POST /api/nutrition/food` (`entry.toJson()`) → backend `nutritionController.addFood` → `nutritionService.addFood` (new id, `setLog`, Firestore write under `users/{uid}/foodLogs`) → `201 { entry }` → on success `ref.invalidate(nutritionDayProvider)` → `GET /api/nutrition/today?date=` → `nutritionService.getDay` → `groupByMeal` → `{ dateKey, entries, meals, totals }` → `NutritionDay.fromJson` → screen re-renders meal sections + `_MacroSummary`.

### 7.1 Root cause (confirmed in code) — backend `getDay` drops `mealId`/`mealName` on read

The write side is correct and persists the meal association; the **read** side strips it:

- `frontend/lib/features/.../add_food_sheet.dart` `_buildEntry()` **correctly** sets `mealId: widget.mealId` and `mealName: widget.mealName` on both the selected-food and custom-food paths.
- `frontend/lib/models/food.dart` `FoodLogEntry.toJson()` emits `mealId`/`mealName` when non-null, and the backend `foodLogUpsertSchema` accepts them, so `nutritionService.addFood` writes them into `users/{uid}/foodLogs`. **The association is stored.**
- The defect: `backend/src/services/nutritionService.ts` `getDay()` maps each Firestore row into a `FoodLogEntry` containing only `{ id, mealType, name, quantity, calories, protein, carbs, fat, fiber, sugar, sodium }` — it **omits `mealId` and `mealName`**. `backend/src/utils/nutritionCalc.ts` `FoodLogEntry` interface has no `mealId`/`mealName` fields either, and `groupByMeal` keys only by `mealType`.
- Consequence: `GET /api/nutrition/today` returns `entries[]` with `mealId:null, mealName:null` for every log. On the client, `NutritionDay.entriesForCustomMeal(meal)` matches `e.mealId == meal.id || (e.mealId == null && e.mealName == meal.name)`. With both null, a food added to a **named custom meal** matches no section and vanishes; `entriesFor(MealType.custom)` only returns entries with empty `mealName`, so it won't catch it either. This is exactly the reported "added food does not appear."

This root cause is confirmed by direct read of all four files (`nutritionService.ts`, `nutritionCalc.ts`, `food.dart`, `add_food_sheet.dart`); it is not a runtime hypothesis.

### 7.2 The fix (concrete)

1. `backend/src/utils/nutritionCalc.ts`: add `mealId?: string | null;` and `mealName?: string | null;` to the `FoodLogEntry` interface. `groupByMeal` stays keyed by `mealType` for the per-meal `meals` summary (built-in meal sections are unaffected), but the fields now survive on each entry object.
2. `backend/src/services/nutritionService.ts` `getDay()`: carry the fields through the row mapping — `mealId: (r.mealId as string) ?? null, mealName: (r.mealName as string) ?? null`. The top-level `entries[]` (which the Flutter `NutritionDay` model consumes for custom-meal grouping) now retains the association, so `entriesForCustomMeal` matches again.
3. No frontend write-path change is needed for the appear-in-meal symptom; the write already persists `mealId`/`mealName`.

### 7.3 Date-key handling (confirmed non-issue, documented)

Verified symmetric: `nutritionDateProvider` builds the local-time key once; `AddFoodSheet` is constructed with `dateKey` and sends `widget.dateKey` on the add; `getDay`/`nutritionDayProvider` read the same provider value; backend `listByDate` queries `where('dateKey','==',dateKey)` and only substitutes `todayKey()` for an **absent read** date, never on writes. So add and read use the same key by construction. The sole residual risk is the user crossing midnight with the sheet open — a minor edge case, **not** the reported bug. To close even that edge case the implementer captures `dateKey` once at sheet open (`initState`) rather than re-reading the provider at save time. This is explicitly a low-priority hardening step; §7.1 is the actual bug and is fixed first.

### 7.4 Idempotency on multiple taps (chosen approach: client-supplied id)

There is currently no idempotency: `foodLogUpsertSchema` has no `id`, `nutritionService.addFood` always calls `nutritionRepository.newLogId(uid)`, and `FoodLogEntry.toJson()` omits `id`. The `_saving` guard blocks double-taps within one sheet, but a slow-network re-tap can still double-write. **Decision: adopt client-supplied-id idempotency** (not merely the in-flight guard), because acceptance item 23 requires that multiple adds for the same action never duplicate, and the in-flight guard alone does not cover slow-network re-taps. The exact, minimal additive changes:

- `backend/src/validators/nutritionValidators.ts`: add `clientId: z.string().min(1).max(200).optional()` to `foodLogUpsertSchema`.
- `backend/src/services/nutritionService.ts` `addFood`: `const id = input.clientId ?? nutritionRepository.newLogId(uid);` then `setLog(uid, id, ...)`. `setLog` already uses `set(..., { merge: true })`, so this is create-if-absent / update-if-present — a re-tap with the same id updates the one record instead of creating a second. `clientId` is not persisted as a field (it only drives the doc id).
- `frontend/lib/models/food.dart`: include the id in the add payload (send `clientId`), and `_AddFoodSheetState` generates one stable id per sheet open (e.g. in `initState`, a UUID or `DateTime.now().microsecondsSinceEpoch`-based key), reused across re-taps of that sheet instance.
- Keep the existing in-flight `_saving` button-disable as a first line of defense; the client id is the correctness guarantee.

### 7.5 Loading / success / failure states

`nutritionRepository.addFood` returns `Err(ServerFailure)` on a missing `entry`; a backend `422` (e.g. calories out of range) surfaces as a SnackBar. The implementer must confirm all three UI paths are visibly handled: loading spinner while saving, success → invalidate `nutritionDayProvider` **before** pop then close, failure → SnackBar with the backend message. These are verified, not redesigned.

### 7.6 Tests for the bug

- **Backend round-trip test:** write a log with `mealType:'custom', mealId:'m1', mealName:'Pre-workout'`, call `getDay`, assert the returned entry still carries `mealId:'m1'` and `mealName:'Pre-workout'`.
- **Backend idempotency test:** call `addFood` twice with the same `clientId`; assert exactly one `foodLogs` doc exists and totals are not doubled.
- **Flutter widget test:** add a food to a named custom meal; after refresh assert it renders in that meal's section and the daily macro summary includes it.

The fix must satisfy every acceptance item in §9.7.

---

## 8. Testability

- **Backend unit (vitest, mock Firestore like `userRepository.test.ts`):** `roleService.resolveRole` (role present / absent+trainer / absent → student / legacy); `assertTrainerOwnsStudent` (owned → ok, other trainer → 404, missing → 404, inactive → 404); `link-trainer` (valid → link + increment once, invalid → soft 200, duplicate → idempotent no double-count, trainer-as-student → 409); referral-code normalization.
- **Backend integration (supertest-style over the app):** every trainer route for 401 (no token), 403 (student token), 404 (unowned student), 200 (owned). Nutrition add→getDay round-trip under a fixed dateKey; idempotent add with a client-supplied id.
- **Firestore rules (emulator):** trainerLinks/trainers/referralCodes not client-readable/writable; cross-student denial; unauth denial.
- **Flutter widget tests:** AddFoodSheet add→list refresh (overriding providers with fakes); register-with-invalid-code shows the message and still lands as student; router sends `role:trainer` to the trainer shell and `role:student` to `HomeShell`.
- **Hard-to-test seams noted:** signed-URL generation and real Storage/Firestore are integration-only; abstract behind the repository so services are unit-testable with fakes. Role middleware is tested by injecting a fake role resolver.

Verification gate commands (run in `backend/`): `npm run typecheck`, `npm run lint`, `npm test`. In `frontend/`: `flutter analyze`, `flutter test`.

---

## 9. Acceptance criteria

### 9.1 Roles & routing
1. A new `role` field exists on `users/{uid}`; `POST /api/auth/verify` returns the resolved role; the Flutter router routes `trainer` → trainer shell and `student` → existing `HomeShell`, using backend-returned role only.
2. A legacy account with no `role` and no `trainers/{uid}` doc resolves to `student` and sees the unchanged student app.
3. The seeded trainer (`dreamphysics`) signs in with normal Firebase email/password via "Trainer Login" and lands on the Trainer Dashboard.

### 9.2 Registration & linking
4. Registration keeps Name/Email/Password/Confirm plus ONE optional field with placeholder exactly `Enter trainer referral code (optional)`.
5. Empty code → a normal independent student with the exact existing experience (no trainer link created).
6. Code `dreamphysics` (any case/whitespace) → student created AND `trainerLinks/{studentUid}` created with `trainerId` of the Dream Physics trainer, `profile.trainerId` mirrored, `totalStudents` incremented exactly once; the student appears in that trainer's list automatically.
7. Invalid code → account still created as a student; UI shows an "Invalid trainer referral code" message; no link created.
8. The relationship is stored by `trainerId`/`studentUid` IDs, never by email, and is validated server-side (client never writes the link).

### 9.3 Trainer views
9. Trainer Dashboard shows Total/Active students, Today's Activity, Workout Sessions, Students needing attention, Recent activity, and a student list with working search, sort, and the filters All/Active/Inactive/Recently active/Goal/Progress.
10. Each student card shows photo, name, goal, current weight, progress summary, last workout, today's activity status, nutrition status, overall progress.
11. Student detail has Overview, Workout, Workout History, Nutrition, Water, Progress, and Progress Photos tabs, each populated from the student's existing data (no mock data), with all fields listed in requirement 6 present when the underlying data exists.
12. Nutrition tab shows calories/protein/carbs/fat/fiber/sugar/sodium, meals, food items, meal timing, daily totals, and historical days (reusing `getDay`).

### 9.4 Security
13. A trainer calling any `/api/trainer/students/:studentUid/*` for a student not linked to them receives `404`; for a student they own, `200`.
14. A student (role=student) calling any `/api/trainer/*` receives `403`; an unauthenticated call receives `401`.
15. One trainer can never read another trainer's students or links; `trainers`/`trainerLinks`/`referralCodes` are not client-readable or client-writable per `firestore.rules`; no student data is publicly readable.
16. Progress photos are visible to a trainer only when `shareProgressWithTrainer` is true, served via short-lived signed URLs; Storage rules remain owner-only and are not loosened.

### 9.5 Student ⇄ trainer two-way
17. A linked student sees "My Trainer" with trainer name, photo, status (and contact if shared), e.g. "Dream Physics / Referral Code: dreamphysics"; an unlinked student sees no trainer / an invite to add one.

### 9.6 Non-regression
18. With no referral code, every existing student feature (Home, Workout, Nutrition, AI Coach, Profile, Water, Routine, Alarm, Progress, History, Analytics, Notifications) works exactly as before; the Midnight Energy bottom-nav shell is unchanged.
19. `npm run typecheck`, `npm run lint`, `npm test` (backend) and `flutter analyze`, `flutter test` (frontend) all pass with no new errors.

### 9.7 Nutrition Add-Food bug
20. Adding a food (search-selected or custom) to a built-in or custom meal makes it appear immediately in the selected meal.
21. The meal total and the daily calories/protein/carbs/fat (+fiber/sugar/sodium) update to include it.
22. The entry persists across leaving and reopening the Nutrition screen and is readable in the trainer's Nutrition tab for that student/date.
23. Tapping "Add food" multiple times for the same action does not create duplicate records.
24. Loading, success, and failure states are each handled visibly (spinner / refresh+close / error message).

---

## 10. Out of scope
- Trainer-initiated writes to student data (coaching notes, assigning workouts/plans) — read/monitor only.
- Trainer-to-student or in-app messaging/chat.
- Multiple trainers per student, or trainer teams/orgs.
- Admin UI for creating trainers (bootstrapped via seed script; future admin tooling noted).
- Per-photo (vs. per-student) progress-photo sharing granularity.
- Payment/subscription/billing for trainers.
- Changing or redesigning any existing student screen beyond the additive referral field, "Trainer Login" action, and "My Trainer" card.

---

## 11. Files to add / modify (implementation map)

**Backend — add:** `src/routes/trainer.routes.ts`, `src/controllers/trainerController.ts`, `src/services/trainerService.ts`, `src/services/roleService.ts`, `src/repositories/trainerRepository.ts`, `src/repositories/trainerLinkRepository.ts`, `src/validators/trainerValidators.ts`, `src/scripts/seedTrainer.ts`, tests in `src/__tests__/trainer*.test.ts`.
**Backend — modify:** `src/app.ts` (mount `/api/trainer`, `/api/student`), `src/routes/auth.routes.ts` (+`/link-trainer`), `src/controllers/authController.ts` + `src/services/authService.ts` (return role; link endpoint), `src/repositories/userRepository.ts` (`getRole`, preserve `role` on ensure), `src/middleware/auth.ts` or new `src/middleware/role.ts` (`requireTrainer`), `src/validators/profileValidators.ts` (+`trainerId`, `shareProgressWithTrainer`), `src/services/progressPhotoService.ts` (signed-URL for authorized trainer), `package.json` (`seed:trainer`). Nutrition bug fix (§7): `src/utils/nutritionCalc.ts` (+`mealId?`/`mealName?` on `FoodLogEntry`), `src/services/nutritionService.ts` (`getDay` carries `mealId`/`mealName`; `addFood` uses `clientId`), `src/validators/nutritionValidators.ts` (+optional `clientId`).
**Firebase — modify:** `firestore.rules` (deny trainer/link/code collections to clients), `firestore.indexes.json` (`trainerLinks` composite). `storage.rules` unchanged.
**Flutter — add:** `lib/features/trainer/**` (data/presentation/providers, shell, dashboard, students, detail with tabs, progress, profile), trainer models.
**Flutter — modify:** `lib/features/auth/data/auth_repository.dart` (`role` on `AccountInfo`, `linkTrainer` method + `linkTrainerResultProvider`), `lib/features/auth/presentation/register_screen.dart` (optional code field, calls `linkTrainer` after register), `lib/features/auth/presentation/verify_email_screen.dart` (surfaces the link result / invalid-code message), `login_screen.dart` ("Trainer Login"), `lib/core/routing/app_router.dart` + `route_names.dart` (role branch inserted **before** the onboarding gate, `trainerHome` route), `lib/features/profile/presentation/profile_screen.dart` ("My Trainer" card + sharing toggle). Nutrition bug fix (§7): `lib/models/food.dart` (`clientId` in add payload), `add_food_sheet.dart` (stable per-sheet id in `initState`, invalidate-before-pop).

---

## 12. Responses to design-review findings

Each finding from `design-review.json` is addressed below. All blocking (HIGH/MEDIUM) findings are resolved in the design; the choices align with the original requirements (server-trusted role, additive layer, no public student data, no broken existing features).

- **#1 (HIGH) — Nutrition root cause misattributed.** Resolved. §7 is rewritten: the root cause is now identified as the backend `getDay` row mapping and the `nutritionCalc.FoodLogEntry` interface dropping `mealId`/`mealName` on read (confirmed by direct read of `nutritionService.ts`, `nutritionCalc.ts`, `food.dart`, `add_food_sheet.dart`). The frontend `_buildEntry`/`toJson` are confirmed correct. The concrete two-line fix (§7.2) plus the backend round-trip test and the named-custom-meal widget test (§7.6) are specified.
- **#2 (HIGH) — Router branch contradiction + role not in verify response.** Resolved. (a) §6.1 now places the role branch **before** the onboarding gate with the exact `app_router.dart` insertion point and code, so a seeded trainer (`onboardingCompleted:false`) routes to the trainer shell and never hits student onboarding. (b) §5.1 pins the wiring: `verifyAndSync` injects `roleService.resolveRole(uid)` into the returned account; `AccountInfo.fromJson` parses `account['role'] ?? 'student'`; the redirect reads only `account.role`.
- **#3 (MEDIUM) — Referral-link call timing / invalid-code message.** Resolved. §6.4 pins the call site: `linkTrainer(code)` fires right after a successful `register()` on the fresh token (route needs only `authenticate`), the result is stored in a `linkTrainerResultProvider`, and the invalid-code / success message is surfaced on the **verify-email screen** (which is mounted after registration), not the torn-down register screen.
- **#4 (MEDIUM) — Add-Food idempotency under-specified.** Resolved. §7.4 chooses the client-supplied-id approach and specifies every change: optional `clientId` in `foodLogUpsertSchema`, `addFood` using it via the already-`merge` `setLog`, the client generating one stable id per sheet open, and the in-flight guard retained as a secondary defense.
- **#5 (MEDIUM) — Date-key mislabeled highest-likelihood.** Resolved. §7.3 downgrades date-key to a confirmed non-issue (add/read use the same key by construction) and an optional midnight-edge-case hardening (capture `dateKey` at sheet open); §7.1 is sequenced as the actual fix.
- **#6 (MEDIUM) — Signed-URL credential constraint.** Resolved. §2.3 now states the V4 signed-URL path requires key-bearing credentials, retains per-photo graceful `url:null` on signing failure, and forbids CI/emulator tests that assert real signed URLs.
- **#7 (NIT) — Non-active link handling.** Resolved. §5.1 states any non-`active` status yields 404 for per-student endpoints; §5.3 states `GET /api/student/trainer` treats non-`active` as "no trainer" (`{ trainer: null }`).
- **#8 (NIT) — totalStudents source.** Resolved. §5.3 returns a live count query from `trainerLinks` as authoritative for display; §3.3 marks the stored counter as convenience-only.
- **#9 (NIT) — Composite index precision.** Resolved. §3.6 specifies exactly one composite index `trainerId ASC, status ASC, updatedAt DESC`.
- **#10 (NIT) — Legacy role write-back.** Resolved. §2.5 states `resolveRole` is purely computed per verify and never writes `role` back, preserving the no-write-on-unchanged-reopen invariant.
