# Design Review — Dynamic Multi-Trainer Referral System (FitTrack AI)

Reviewed document: `.agents/tasks/dynamic-multi-trainer-referral/design.md` (revision pass responding to a prior CHANGES_REQUESTED verdict of 0 HIGH / 4 MEDIUM / 5 NIT).
Grounding sources read in full: the design, the hardcoded-touchpoint `inventory.md`, and the actual backend/firebase/frontend source listed below. This review was performed fresh, without the context that produced the design; every structural claim the design makes about the existing code was checked against source.

## Method

I read and cross-checked the following against the design's claims:

- `backend/src/repositories/trainerRepository.ts`
- `backend/src/repositories/trainerLinkRepository.ts`
- `backend/src/services/trainerService.ts`
- `backend/src/services/studentService.ts`
- `backend/src/services/roleService.ts`
- `backend/src/services/nutritionService.ts`
- `backend/src/scripts/seedTrainer.ts`
- `backend/src/validators/trainerValidators.ts`
- `backend/src/routes/auth.routes.ts`, `backend/src/routes/student.routes.ts`
- `firebase/firestore.rules`, `firebase/firestore.indexes.json`
- `frontend/lib/features/trainer/presentation/trainer_profile_screen.dart`
- A repo-wide grep for every reader/writer of `totalStudents`.

The design is a careful revision; its eight required design decisions are each resolved with a concrete rule and justified against the real architecture. Most of its claims about the existing code are accurate. The findings below are the residual gaps.

---

## Findings

### 1. NIT — `_ReferralCard` is render-gated on a non-empty code, so a code-less trainer cannot reach the "Generate" affordance

Where: `§5.2`, cross-checked against `frontend/lib/features/trainer/presentation/trainer_profile_screen.dart`.

The design puts the new "Change / Generate New" (and custom-code) affordance *inside* `_ReferralCard`, and argues the card is "never empty" because registration always generates a code (`§4.3 ensureTrainerAccount`). But the current Profile screen renders the card only when a code exists:

```dart
if (profile.referralCode != null && profile.referralCode!.isNotEmpty)
  _ReferralCard(code: profile.referralCode!),
```

The design's own edge case `§5.7` ("Trainer registration call fails after the Firebase user is created … retryable 'Couldn't finish trainer setup'") describes exactly the state where a trainer account exists with no code. In that state `_ReferralCard` does not render, so there is no in-Profile path to generate a first code — the user is told to retry registration, but the Profile has no fallback affordance. This also conflicts with the requirement's "if the trainer skips code creation during onboarding, they should be able to create it later from Profile."

Concrete fix: specify that the Profile referral section renders unconditionally when the account is a trainer. When `referralCode` is null/empty, show a "Create referral code" state (Generate + custom-code options) instead of hiding the card. Example rule for `§5.2`: "Render the referral section for every trainer. If `TrainerProfile.referralCode` is null/empty, the card body is the create-code affordance (Generate / Choose custom); otherwise it is the display + Change affordance."

### 2. NIT — `GET /api/student/trainer` response shape does not carry a `MyTrainer.none` discriminant; `§5.3` assumes one

Where: `§5.3` / `§5.5`, cross-checked against `backend/src/services/studentService.ts`.

`studentService.getTrainer` returns `{ trainer: null }` for the no-active-link case and `{ trainer: {…} }` otherwise. `§5.3` describes a `MyTrainer.none` state driving a "Connect to Trainer" affordance, and `§5.5` says "`TrainerProfile`/`MyTrainer` are reused." The mapping from `{ trainer: null }` to a `MyTrainer.none` sentinel is assumed but never stated, and the backend contract for the new connect-from-profile flow (which route returns what on success) is only described as "same body/semantics as `link-trainer`."

Concrete fix: in `§5.5`, state the client parse rule explicitly — `{ trainer: null }` → `MyTrainer.none`, `{ trainer: {…} }` → `MyTrainer.connected(...)` — and confirm `GET /api/student/trainer` is unchanged (it is; no backend change needed here). This is documentation-level; no code claim is wrong.

### 3. NIT — `registerTrainerSchema` takes `email`/`photoUrl` "from the verified token where possible", but the token's available fields are not pinned

Where: `§4.6` (trainer registration) and `§4.5` (`register-trainer`).

The design says `register-trainer` "sets the role server-side from the verified token's UID" and that `email`/`photoUrl` are "taken from the verified token where possible." Firebase ID tokens reliably carry `uid` and usually `email`, but `email` can be absent (e.g. some providers) and there is no photo on an email/password token. The design does not state what happens when the token lacks an email — whether the trainer doc stores `email: null`, or the client must supply it. The USER path stores profile fields separately; the trainer doc's `email` provenance is left implicit.

Concrete fix: pin the rule in `§4.6` — e.g. "`email` is read from the decoded token (`req.user.email`); if absent, the trainer doc stores `email: null`. `photoUrl` always defaults to `null` at registration (deferred per `§8`). The client never asserts either." This keeps role/identity server-trusted and removes the "where possible" ambiguity.

---

## Verified Assumptions (checked against source — correct)

1. **`roleService.resolveRole` is purely computed and never writes `role` back** (`§1`, `§4.5`, `§6`). Confirmed in `roleService.ts`: returns stored role if set, else `trainer` iff `trainers/{uid}` exists, else `student`; no write path. Trainer self-registration creating `trainers/{uid}` + `users/{uid}.role='trainer'` will resolve to `trainer` on the next verify. **Decision (6) server-trusted role: satisfied.**

2. **The link relationship is keyed by `studentUid` → `trainerId`, never by code or email** (`§2.2`, `§2.4`, `§3.3`). Confirmed in `trainerLinkRepository.ts` (`trainerLinks/{studentUid}` with a `trainerId` field) and `trainerService.linkStudent`. This is the crux that makes code change (decision 2) and migration (decision 4) non-destructive: retiring/normalizing a code cannot touch any link. **Decisions (2) and (4): the safety argument is sound.**

3. **There is no `trainerLinkRepository.create`; the link is created inline via `tx.set(linkRef, …)`** (`§4.4`, review response #6). Confirmed — the repository exposes only `get`, `listByTrainer`, `doc`, `fieldValue`, and `linkStudent` builds the doc inline inside its transaction. The revised wording is accurate.

4. **The stored `totalStudents` counter is advisory; `getProfile` overwrites it with the live `countActiveStudents` query, and nothing else reads it** (`§2.3`, `§4.4`). Verified by grep: `totalStudents` is *written* by `incrementTotalStudents`, the inline increment in `linkStudent`, and the seed; it is *read for display* only in `getProfile`, which returns `totalStudents: activeStudents`. Therefore the switch path writing no counter cannot cause a visible drift, and "no decrement can go negative" holds. **Decision (3) switch-vs-reject, counter correctness: satisfied.**

5. **`linkStudent` today has no switch guard and merges/re-links** (`§2.3`, `§4.4`). Confirmed: current `linkStudent(studentUid, referralCode)` has no `confirmSwitch` param and `LinkResult.reason` is only `'invalid_code'`. The design's added `already_linked` branch and `confirmSwitch` round-trip are a true extension, not a duplication. The "never silent overwrite" requirement is met because the second connection to a different trainer returns `already_linked` with no write unless `confirmSwitch===true`.

6. **`referralCodes/{codeLower}` is a lookup index; the `trainers/{trainerId}` record is authoritative** (`§3.2`, `§4.8`). Confirmed: `findTrainerIdByReferralCode` reads only `trainerId` from the code doc, and the connect/switch flow resolves and loads the trainer record. The design adds an explicit `§4.4` step 2 ("verify the resolved account is actually a trainer" via `roleService` + `trainers/{trainerId}` exists + active) so a dangling or non-trainer code doc yields `invalid_code`. **Decision (5): satisfied.**

7. **Firestore rules are fail-closed for `trainers`/`referralCodes`/`trainerLinks` and need no change** (`§3.5`, `§4.8`, `§9`). Confirmed in `firestore.rules`: all three are `allow read, write: if false` plus a trailing default-deny; backend Admin SDK bypasses rules. The composite index `trainerLinks (trainerId ASC, status ASC, updatedAt DESC)` exists in `firestore.indexes.json`, and code lookups are single-document `get`s needing no index — so "no index change" is correct. **Decision (7): satisfied; referral codes grant no API access because every `/api/trainer/*` route is still gated by `authenticate`+`requireTrainer`+`assertTrainerOwnsStudent`.**

8. **The nutrition Add-Food fix is not touched** (`§9`). Confirmed in `nutritionService.ts`: `getDay` maps `mealId: (r.mealId) ?? null, mealName: (r.mealName) ?? null`, and `addFood` destructures `const { clientId, ...data } = input` so `clientId` is the doc id and never persisted. No file the design modifies overlaps the nutrition read/write path. **Decision (8) no nutrition regression: satisfied.**

9. **The legacy seed is the single hardcoded `dreamphysics` touchpoint** (`§2.4`, `§6`). Confirmed in `seedTrainer.ts`: `REFERRAL_CODE='dreamphysics'`, `TRAINER_NAME='Dream Physics'`, `SEED_TRAINER_UID`, and the three writes (`users/{uid}.role`, `trainers/{uid}`, `referralCodes/dreamphysics → { trainerId }`). The migration must backfill `code`/`active`/`createdAt`/`updatedAt` onto the `referralCodes/dreamphysics` doc (which today stores only `{ trainerId }`) — the design's `§2.4` says exactly this. **Decision (4) safe migration: well-grounded; nothing deleted, no duplicate, links untouched.**

10. **Custom-code support is practical and the custom-code schema (length 4–20, `^[A-Za-z0-9]+$`) is a strict subset of the connect schema (min 3 / max 40)** (`§2.1`, `§4.6`). Confirmed `linkTrainerSchema` is min 3 / max 40 lowercase; a 4–20 custom/generated code always validates on the connect path. **Decision (1) custom-vs-generated: resolved as "both, generated-default + optional custom," justified against the existing single-doc-per-code index.**

11. **The generated-code transaction shape is Firestore-legal** (`§2.5`). The pseudocode places all `tx.get`s before any `tx.set`/`tx.update`, uses an outer candidate loop (candidate generated outside the transaction), aborts a colliding attempt with zero writes, and bounds retries at N=5 → `503`. This is correct Firestore transaction discipline and resolves the prior #2 concern.

12. **The reserved-list vs migration tension is resolved** (`§2.4`, `§4.6`, review response #3). The migration backfills via Admin SDK *without* running the format/reserved validator, grandfathering `dreamphysics`; the reserved list applies only to new claims so no *other* trainer can re-claim `dreamphysics` after the legacy owner moves off it. Internally consistent.

---

## Unverified / Wrong Assumptions

No design claim about the existing code was found to be wrong. The following are assumptions the design makes that this review could not fully verify from source, flagged for the implementer (none rises to a blocking finding on its own):

1. **`req.user` token fields (`email`, name) available to `register-trainer`.** The design assumes the verified token exposes `email`/name to populate the trainer doc. I read `auth.routes.ts` (confirms `authenticate` runs first) but did not open `middleware/auth.ts` to confirm exactly which decoded-token fields are attached to the request. This underlies Finding 3; the implementer should confirm the attached shape before relying on `email`.

2. **`RegisterTrainerController` on the root `ProviderContainer` behaves like `LinkTrainerController`.** `§5.1` asserts the existing `linkTrainerControllerProvider` survives the register-screen teardown and that a new controller can reuse that exact pattern. The inventory corroborates `LinkTrainerController` exists and owns the request across teardown, but I did not open `auth_controller.dart` to confirm it is mounted on a root container (vs the screen's scope). Low risk — the inventory explicitly lists it as a reuse point — but it is an assumption, not a verified fact, in this review.

3. **Migration display-case recovery.** `§2.4` says the `referralCodes` backfill sets `code` = "original display case if derivable else the normalized value." For the legacy `dreamphysics` doc, the display case is recoverable from `trainers/{uid}.referralCode` (`'dreamphysics'`), so this is fine in practice; for any hand-created code docs lacking a trainer-side display form the fallback to normalized value is a stated, acceptable degradation. Not verifiable beyond the known seed, but harmless.

---

## Verdict basis

HIGH findings: 0. MEDIUM findings: 0. NIT findings: 3.

Per the mechanical rule (HIGH+MEDIUM > 0 → CHANGES_REQUESTED; else APPROVED), the count of HIGH+MEDIUM is zero, so the verdict is **APPROVED**. The three NITs are documentation/UI-robustness clarifications that do not block implementation but are worth folding in: render the Profile referral section even when a trainer has no code yet (Finding 1), state the `{trainer:null}`→`MyTrainer.none` mapping (Finding 2), and pin the token-field provenance for trainer registration (Finding 3).
