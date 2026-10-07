# Inventory — Fixed Single-Trainer (`dreamphysics`) Referral Logic

Status: inventory only. **No code changes made in this step.** This document catalogs every touchpoint of the current fixed single-trainer referral design so the dynamic multi-trainer referral system can evolve it without duplicating or regressing existing work.

Scope note: the repo root (`c:\Users\cheta\OneDrive\Desktop\fit`) is NOT a git repo. There are two independent git repos (`backend/`, `frontend/`, both on branch `feat/trainer-student-2way`) plus an unversioned `firebase/` directory. The prior-task artifacts (`.agents/tasks/trainer-student-2way/{design.md,plan.md,FEAT-00x}`) describe the architecture being evolved and were read first.

Legend for "must become":
- **HARDCODE** = a fixed single-`dreamphysics` touchpoint that must be removed/generalized.
- **REUSE** = dynamic-capable infrastructure that already exists and should be evolved in place, not rebuilt.
- **DO-NOT-REGRESS** = a prior bug fix that must keep working.

---

## Summary count

Total hardcoded `dreamphysics` / "Dream Physics" / single-trainer touchpoints found: **12** across production code + config, plus **5** test fixtures that encode the single seeded trainer.

- **Backend production/config: 3** real hardcoded touchpoints — `src/scripts/seedTrainer.ts` (the entire fixed-trainer seed: `REFERRAL_CODE='dreamphysics'`, `TRAINER_NAME='Dream Physics'`, `SEED_TRAINER_UID` env, the three writes), the `seed:trainer` npm script in `package.json`, and the single-code comment in `src/validators/trainerValidators.ts`.
- **Backend tests: 4 files** hardcode `trainer-1` / `dreamphysics` fixtures (`trainerService.test.ts`, `trainerRoutes.test.ts`, `roleService.test.ts`, and references in `trainerService.test.ts`). These are fixtures, not production logic.
- **Frontend production: 0** hardcoded `dreamphysics` touchpoints — the client never contains the literal code; `referralCode` is read dynamically from the trainer profile / link response.
- **Frontend tests: 3 files** use `'Dream Physics'` / `'dreamphysics'` as fixture data (`trainer_routing_test.dart`, `trainer_dashboard_test.dart`, `register_referral_test.dart`).
- **Firebase: 0** hardcoded `dreamphysics` — rules/indexes are already generic over `trainers`, `referralCodes`, `trainerLinks`.

The single source of the fixed `dreamphysics` trainer is the **seed script** (`backend/src/scripts/seedTrainer.ts`). Everything else in production is already dynamic (code→trainer lookup via `referralCodes/{codeLower}`, trainer resolved by UID). The dynamic model primarily needs: (1) trainer self-registration + referral-code creation/change endpoints, (2) removal/retirement of the single-trainer seed, and (3) a frontend account-type selector + trainer referral-code management UI.

---

## BACKEND (`c:\Users\cheta\OneDrive\Desktop\fit\backend`, branch `feat/trainer-student-2way`)

### (a) Hardcoded / special-cased single-`dreamphysics` trainer

| File + symbol | What it does today | Must become (dynamic multi-trainer) |
|---|---|---|
| `src/scripts/seedTrainer.ts` — consts `REFERRAL_CODE = 'dreamphysics'`, `TRAINER_NAME = 'Dream Physics'`; reads `process.env.SEED_TRAINER_UID`; writes `users/{uid}.role='trainer'`, `trainers/{uid}` (`referralCode`/`referralCodeLower='dreamphysics'`), `referralCodes/dreamphysics → {trainerId}` | The ONLY hardcoded trainer datum. Seeds a single fixed trainer with a fixed code from one env UID. | **HARDCODE → remove the single-trainer bootstrap.** Trainers are created dynamically via a trainer-registration flow (TRAINER account type) that creates `users/{uid}.role='trainer'` + `trainers/{uid}` and generates/assigns a unique `referralCode`. Keep an optional generic dev-seed only if useful, but it must not be the sole/privileged trainer and must not hardcode `dreamphysics`. Remove `SEED_TRAINER_UID` as the single-trainer mechanism. |
| `package.json` — script `"seed:trainer": "tsx src/scripts/seedTrainer.ts"` | Runs the fixed-trainer seed. | **HARDCODE →** drop or repurpose once trainer self-registration exists. Not a privileged bootstrap. |
| `src/validators/trainerValidators.ts` — doc comment: "the only seeded code today is 'dreamphysics'" | Comment only; `linkTrainerSchema.referralCode` is already generic (trim/lowercase, min/max). | **HARDCODE (comment) →** update wording; the schema itself is reusable as-is for dynamic codes. |

### (b) Dynamic-capable infrastructure to REUSE / evolve (NOT duplicate)

| File + symbol | Current behavior (already dynamic) | How to reuse/evolve |
|---|---|---|
| `src/services/roleService.ts` — `RoleService.resolveRole(uid)` | `users/{uid}.role ?? (trainers/{uid} exists ? 'trainer' : 'student')`. Purely computed, never writes `role` back. | **REUSE unchanged.** Already supports unlimited trainers (any UID with a `trainers/{uid}` doc resolves to trainer). Trainer self-registration simply creates that doc. |
| `src/services/authService.ts` — `verifyAndSync` injects resolved `role` into the account | Server-trusted role on every `/api/auth/verify`. | **REUSE unchanged.** |
| `src/repositories/trainerRepository.ts` — `get`, `upsert`, `findTrainerIdByReferralCode(codeLower)`, `setReferralCode(codeLower, trainerId)`, `countActiveStudents`, `incrementTotalStudents` | Generic O(1) code→trainer lookup via `referralCodes/{codeLower}`; trainer profile upsert; live active-student count from `trainerLinks`. Already multi-trainer. | **REUSE + extend.** `setReferralCode` + `upsert` are the building blocks for create/change-code. For "change code" add logic to write the new `referralCodes/{newCode}` doc and retire the old one WITHOUT touching `trainerLinks` (relationships survive). Enforce case-insensitive uniqueness via the existing single-doc-per-code index. |
| `src/repositories/trainerLinkRepository.ts` — `trainerLinks/{studentUid}` get/create/`listByTrainer` | Authoritative one-student→one-trainer link keyed by `studentUid`. | **REUSE unchanged.** Already the ID-based relationship key (never email). Supports multiple trainers via `trainerId` field. |
| `src/services/trainerService.ts` — `linkStudent(studentUid, referralCode)` | Normalizes code, dynamic `findTrainerIdByReferralCode`, soft `{linked:false,'invalid_code'}` on unknown, transactional link + profile mirror + idempotent `totalStudents++`, `409` if a trainer tries to link as student. Plus `getProfile`, `listStudents`, per-student `overview/workouts/history/nutrition/water/progress/photos`, `assertTrainerOwnsStudent` wrapper. | **REUSE as the core linking engine.** Already fully dynamic (any valid code → its owning trainer). For "one user → one trainer" switching, this is where the already-linked-to-another-trainer confirmation/rejection flow would be added (currently it re-links/merges without a switch guard — see gap below). |
| `src/services/studentService.ts` — `getTrainer`, `setSharing` | `GET /api/student/trainer` returns the linked trainer's public subset (incl. `referralCode`); non-active link → `{trainer:null}`. | **REUSE unchanged.** |
| `src/middleware/role.ts` — `requireTrainer`, `assertTrainerOwnsStudent(trainerId, studentUid)` | Role gate + per-student ownership guard (404 on missing/mismatch/inactive). | **REUSE unchanged.** Works for any trainer UID. |
| `src/routes/auth.routes.ts` — `POST /api/auth/link-trainer` (+ `authController.linkTrainer`) | Authenticated student→trainer link by code; backend-validated. | **REUSE unchanged.** |
| `src/routes/trainer.routes.ts` + `trainerController.ts` (behind `authenticate`+`requireTrainer`) and `src/routes/student.routes.ts` + `studentController.ts` | `/api/trainer/*` and `/api/student/*` slices. | **REUSE + extend.** Add trainer-side endpoints for the dynamic model: create referral code (on onboarding/profile), check-availability (for optional custom codes), change/regenerate code. These belong on the existing `/api/trainer` slice reusing `trainerRepository`. |
| `src/repositories/userRepository.ts` — `getRole(uid)`; `ensureAccount` never writes/clears `role` | Role storage read; no-write-on-unchanged-reopen invariant preserved. | **REUSE unchanged.** |
| `src/validators/profileValidators.ts` — optional `trainerId`, `shareProgressWithTrainer` | Additive profile fields; mirror of the authoritative link. | **REUSE unchanged.** |
| `src/validators/trainerValidators.ts` — `linkTrainerSchema` (+ studentUid/date/history/sharing schemas) | Generic code normalization + validation. | **REUSE + extend** with a code-create/change schema (min/max length, letters+digits, no spaces, reserved-word rejection, case-insensitive) for optional custom codes. |

### (c) Nutrition Add-Food fix — MUST NOT REGRESS

| File + symbol | What it guarantees |
|---|---|
| `src/utils/nutritionCalc.ts` — `FoodLogEntry` has `mealId?: string \| null; mealName?: string \| null;` | Meal association carried on the entry type. |
| `src/services/nutritionService.ts` — `getDay` row mapping sets `mealId: (r.mealId) ?? null, mealName: (r.mealName) ?? null` | **DO-NOT-REGRESS:** `getDay` preserves `mealId`/`mealName` so custom-meal food survives the read path. |
| `src/services/nutritionService.ts` — `addFood` destructures `const { clientId, ...data } = input; const id = clientId ?? newLogId(uid);` and strips `clientId` from the body; `updateFood` voids `clientId` | **DO-NOT-REGRESS:** client-supplied `clientId` idempotency (re-tap writes one doc, no doubled totals; `clientId` never persisted). |
| `src/validators/nutritionValidators.ts` — `clientId` (optional, 1–200), `mealId` (1–80), `mealName` (1–60) nullable/optional | **DO-NOT-REGRESS:** schema accepts the idempotency key + durable meal association. |
| `src/__tests__/nutritionService.test.ts` — tests for `getDay` preserving `mealId`/`mealName`, `clientId` idempotency, no `clientId` persisted | **DO-NOT-REGRESS:** keep these green. |

---

## FRONTEND (`c:\Users\cheta\OneDrive\Desktop\fit\frontend`, branch `feat/trainer-student-2way`)

### (a) Hardcoded / special-cased single trainer

No hardcoded `dreamphysics` in production Dart. The client reads `referralCode` dynamically from backend responses. Test fixtures use `'Dream Physics'`/`'dreamphysics'` as sample data only:
- `test/trainer_routing_test.dart` — `TrainerProfile(... name: 'Dream Physics')` fixture.
- `test/trainer_dashboard_test.dart` — same fixture.
- `test/register_referral_test.dart` — exercises linking a `'dreamphysics'` code and expects `'Connected to Dream Physics'`.

These are **test fixtures** (not production hardcoding). As the dynamic model lands, update them to use arbitrary trainer names/codes rather than the single fixed one.

### (b) Dynamic-capable infrastructure to REUSE / evolve

| File + symbol | Current behavior | Reuse/evolve |
|---|---|---|
| `lib/features/auth/data/auth_repository.dart` — `AccountInfo.role` / `isTrainer` (parsed from backend); `linkTrainer(code)`; `LinkTrainerResult` | Server-trusted role drives shell choice; dynamic link call to `POST /api/auth/link-trainer`. | **REUSE unchanged.** |
| `lib/features/auth/providers/auth_controller.dart` — `LinkTrainerController` / `linkTrainerControllerProvider` | Owns the referral-link request across the register→verify screen teardown; surfaces invalid/connected outcome once. | **REUSE unchanged.** |
| `lib/features/auth/presentation/register_screen.dart` — optional `_referral` field + link-after-register | Optional referral code on registration; empty → normal student. | **REUSE + extend.** Must add the **account-type selector (USER vs TRAINER)** BEFORE account creation. When TRAINER is selected, hide the student referral field and run the trainer-registration flow instead. Keep the existing USER path intact. |
| `lib/features/auth/presentation/verify_email_screen.dart` — surfaces `Invalid trainer referral code` / `Connected to <trainer>` | Non-blocking invalid-code messaging. | **REUSE unchanged.** |
| `lib/features/auth/presentation/login_screen.dart` — Trainer Login (same Firebase email/password; role resolved by backend) | Already role-agnostic sign-in. | **REUSE unchanged.** |
| `lib/features/trainer/data/trainer_models.dart` — `TrainerProfile.referralCode`, `TrainerAssociation.referralCode` | Dynamic referral code carried from backend. | **REUSE unchanged.** |
| `lib/features/trainer/presentation/trainer_profile_screen.dart` — `_ReferralCard` (shows code + Copy) | Displays the trainer's own code with a copy action. | **REUSE + extend.** Add a **Change/Regenerate Code** action (and optional custom-code create/check-availability) wired to the new trainer endpoints. Currently display + copy only — no change flow. |
| `lib/features/profile/presentation/profile_screen.dart` — "My Trainer" card showing `trainer.referralCode` | Student view of their trainer. | **REUSE unchanged.** |
| `lib/features/trainer/providers/trainer_providers.dart`, `trainer_shell.dart`, dashboard/students/detail screens | Role-based trainer shell + per-student views. | **REUSE unchanged.** |

### (c) Nutrition Add-Food fix — MUST NOT REGRESS (frontend side)

| File + symbol | What it guarantees |
|---|---|
| `lib/models/food.dart` — optional `clientId` emitted in `toJson()` | **DO-NOT-REGRESS:** stable idempotency id sent to backend. |
| `lib/features/nutrition/presentation/widgets/add_food_sheet.dart` — stable `clientId` per sheet open + invalidate-before-pop | **DO-NOT-REGRESS:** one id per sheet; refresh before pop; `_saving` guard. |

---

## FIREBASE (`c:\Users\cheta\OneDrive\Desktop\fit\firebase`, UNVERSIONED)

### (a) Hardcoded single trainer

None. No `dreamphysics` literal anywhere in `firestore.rules` or `firestore.indexes.json`.

### (b) Dynamic-capable infrastructure to REUSE / evolve

| File + symbol | Current behavior | Reuse/evolve |
|---|---|---|
| `firestore.rules` — `match /trainers/{trainerId}`, `match /referralCodes/{code}`, `match /trainerLinks/{studentUid}` all `allow read, write: if false;` | Fail-closed client deny; backend Admin SDK only. Generic over any trainer. | **REUSE unchanged.** The deny already covers unlimited trainers; backend writes continue to bypass rules. |
| `firestore.indexes.json` — composite index on `trainerLinks` (`trainerId ASC, status ASC, updatedAt DESC`) | Powers the per-trainer active-student list query. | **REUSE unchanged.** Already keyed by `trainerId` — supports any number of trainers. |

Flag: `firebase/firestore.rules` is unversioned; any change needs user review.

---

## Behavior gaps the dynamic model must close (not "dreamphysics" removals, but required by the new spec)

These are not hardcoded `dreamphysics` touchpoints; they are missing capabilities relative to the dynamic multi-trainer requirements, listed so the next step evolves (not rebuilds) the existing slices:

1. **Account-type selection (USER vs TRAINER) before account creation** — no selector exists in `register_screen.dart` today (student-only registration + optional referral).
2. **Trainer self-registration** — today a trainer only exists via the `seedTrainer.ts` bootstrap. Need a flow that creates `users/{uid}.role='trainer'` + `trainers/{uid}` on registration (no referral code required to become a trainer).
3. **Referral-code generation + uniqueness** — `trainerRepository.setReferralCode` exists but there is no generator or backend uniqueness guarantee beyond single-doc-per-code; need safe generated codes (non-sequential) with backend uniqueness.
4. **Create / change / regenerate code endpoints + UI** — `_ReferralCard` is display+copy only; no create/change endpoint on `/api/trainer/*`. Changing a code must NOT touch `trainerLinks` (existing students stay linked).
5. **Optional custom code** — needs validation (length, charset, no spaces, reserved words, case-insensitive uniqueness) if adopted.
6. **One-user-one-trainer switch guard** — `trainerService.linkStudent` currently re-links/merges; the spec wants a confirm-to-switch (or reject) when already linked to another trainer.

---

Content was rephrased for compliance where prior-task artifacts were summarized.
