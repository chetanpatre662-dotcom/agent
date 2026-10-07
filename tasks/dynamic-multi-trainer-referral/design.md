# Design — Dynamic Multi-Trainer Referral System (FitTrack AI)

Status: revision pass (addresses `design-review.json`, verdict CHANGES_REQUESTED — 0 HIGH, 4 MEDIUM, 5 NIT). Responses to each finding are recorded in §10. Design only — no production code is written in this step.

Revision summary: a single shared claim/availability predicate is now defined and reused by both the claim and availability paths (resolves finding #1). The generated-code generation is pinned to an outer bounded candidate loop where each attempt is one all-reads-before-writes transaction that claims-new + deactivates-old + updates-trainer atomically (#2). The migration is stated to backfill via the Admin SDK without running the format/reserved validator, grandfathering legacy codes including `dreamphysics`, with the reserved list applying only to NEW claims (#3). The switch flow drops stored-counter maintenance entirely and relies on the live `countActiveStudents` the dashboard already uses, so no decrement can go negative (#4). `setReferralCode` is declared superseded (#5), the link creation is described as the existing inline `tx.set` (#6), `status==='active'` is named the single authoritative trainer-active flag with `active` a derived mirror (#7), the `already_linked` response tolerates a null current-trainer name (#8), and the `ensureTrainerAccount` idempotency test is retained (#9).

Scope: **evolve** the already-built trainer-student feature (branch `feat/trainer-student-2way` in both repos) so that trainers are created dynamically through registration and each owns a unique, self-managed referral code, **replacing** the single fixed `dreamphysics` seed. This is a modification of the existing layered backend (`routes → controllers → services → repositories`, Zod validators, `middleware/auth.ts` + `middleware/role.ts`, vitest) and the existing Flutter/Riverpod/go_router/Dio client, plus the unversioned `firebase/firestore.rules` + `firestore.indexes.json`. Do **not** rebuild and do **not** create a parallel system; the inventory at `.agents/tasks/dynamic-multi-trainer-referral/inventory.md` lists every reuse point and the single hardcoded touchpoint (the seed script).

---

## 1. Overview

The prior task already shipped a working, mostly-dynamic trainer layer: `roleService.resolveRole` returns `trainer` for any UID that has a `trainers/{uid}` document (unlimited trainers already supported); `trainerRepository.findTrainerIdByReferralCode(codeLower)` is an O(1) code→trainer lookup over `referralCodes/{codeLower}`; `trainerService.linkStudent` transactionally creates `trainerLinks/{studentUid}` keyed by student UID with a `trainerId` field (never email) and idempotently maintains the student counter; `GET /api/student/trainer`, the full `/api/trainer/*` read surface, `requireTrainer`, and `assertTrainerOwnsStudent` all exist and are role- and ownership-authorized. The one piece of fixed single-trainer logic in the whole system is `backend/src/scripts/seedTrainer.ts`, which bootstraps exactly one trainer with the literal code `dreamphysics`.

The dynamic multi-trainer system is therefore a **targeted extension**, not a redesign. Four capabilities are added and one is removed:

1. **Account-type selection** at registration (USER vs TRAINER) and a **trainer self-registration** path that creates `users/{uid}.role='trainer'` + `trainers/{uid}` server-side (no referral code required to become a trainer).
2. **Referral-code lifecycle for trainers**: a backend generator that guarantees uniqueness, plus `generate / get / change` endpoints and the matching Profile UI (copy + change). Custom codes are supported with an availability check (decision + justification in §2.1).
3. **A one-student→one-active-trainer switch guard** in `linkStudent` so a second connection to a different trainer is never silent (decision in §2.3).
4. **A safe migration** that folds the legacy `dreamphysics` trainer (if seeded) into the ordinary dynamic structure without disconnecting its students, and repurposes the seed script as a generic no-arg bootstrap (decision in §2.4).

The locked stack is unchanged: **TypeScript / Express / Firebase Admin / Zod / vitest** on the backend; **Flutter / Riverpod / go_router / Dio / firebase_auth** on the client; **Firestore + Firebase Storage** for data. No new framework, database, or auth system is introduced. The Midnight Energy theme and all existing student features (including the nutrition Add-Food fix) are preserved; §9 lists the non-regression guarantees.

---

## 2. Design decisions (resolved and justified)

### 2.1 Custom codes vs generated-only → **support both: default-generated, optional trainer-chosen custom code with a backend availability check**

**Decision.** The backend always generates a unique code at trainer creation (so every trainer immediately has a working code and the "skip onboarding" path still yields a code). In addition, a trainer may replace it with a custom code of their choice, validated and reserved server-side via a `Check Availability` affordance.

**Justification.** The existing architecture already makes custom codes cheap and safe:
- `trainers/{trainerId}` already stores both `referralCode` (display form) and `referralCodeLower` (normalized), and `referralCodes/{codeLower}` is already a single-doc-per-code index — case-insensitive uniqueness is a document-existence check, which is exactly what an availability check needs. No new index or data shape is required to support custom codes.
- `linkTrainerSchema` already normalizes (`trim().toLowerCase()`, min 3 / max 40) and the connect path already treats an unknown code as a soft failure, so a custom code flows through the existing linking engine with zero changes to the student side.
- The requirement explicitly asks to support custom codes "if practical within the existing architecture." It is practical, and it is a materially better product (trainers want memorable codes like `ChetanFit`). The only new work is a code-create/change validator and two endpoints that already have a natural home on the `/api/trainer` slice.

The cost is bounded: a reserved-word list and a format validator (letters+digits, no spaces, length bounds). Both are small, pure, and unit-testable. Generated codes remain the default and the fallback, so a trainer who never picks a custom code is fully served. This decision is reversible (dropping to generated-only later is just hiding the custom-code UI), which further lowers the risk of adopting it now.

### 2.2 Old-code behavior on change → **one active code per trainer; the previous code is deactivated (not deleted); existing `trainerLinks` are never touched**

**Decision.** Each trainer has exactly one *active* code at a time. Changing/regenerating the code (a) creates/activates the new `referralCodes/{newCodeLower}` doc pointing at the trainer, (b) marks the previous `referralCodes/{oldCodeLower}` doc `active:false` (retained for audit, not deleted), and (c) updates `trainers/{trainerId}.referralCode` / `referralCodeLower` / `referralCodeUpdatedAt`. `trainerLinks` are **not read or written** by this operation, so every already-connected student stays connected.

**Justification.** The relationship's source of truth is `trainerLinks/{studentUid}.trainerId` — a student's link is keyed to the trainer's UID, never to the code (confirmed in `trainerLinkRepository` and `trainerService.linkStudent`). Because the link does not reference the code at all, retiring a code cannot disconnect anyone; this directly satisfies "existing relationships remain untouched." Keeping the old doc (deactivated) rather than deleting it means a NEW student who types the old code gets a clean, well-defined `invalid`/`inactive` outcome (the connect path checks `active`, §4.4) instead of a dangling pointer, and we retain history for support. Deleting the old doc would also be a correct choice for links, but deactivation is strictly safer (reversible, auditable) at negligible storage cost, so it wins.

### 2.3 Second connection while already linked → **confirmation-driven switch (never silent), server-authorized**

**Decision.** If a student already has an `active` link to trainer A and submits trainer B's code, the connect endpoint does **not** switch silently. It returns a distinct, non-error outcome `{ linked:false, reason:'already_linked', currentTrainer:{ trainerId, name }, requestedTrainer:{ trainerId, name } }`. The client shows the confirmation flow ("You are already connected to A. Switch to B?"). Only when the student explicitly confirms does the client re-call the endpoint with an explicit `confirmSwitch:true` flag; the backend then transactionally updates the link's `trainerId` to B and re-mirrors `profile.trainerId`. The switch is authorized and executed entirely server-side; the client flag only expresses intent, it never carries authority.

**No stored-counter mutation in the switch path.** The switch transaction updates **only** the link row (`trainerId` + `updatedAt`) and the profile mirror. It deliberately does **not** touch either trainer's stored `trainers/{id}.totalStudents`. The reason: the dashboard and `GET /api/trainer/profile` already derive the displayed count from the live query `countActiveStudents(trainerId)` (verified in `trainerService.getProfile`, which overwrites the stored field with the live count — the stored counter is read nowhere else). Introducing a decrement for trainer A would be the first-ever decrement in the system (`incrementTotalStudents` is only ever called with `+1` on new-link creation today) and could drive a drifted stored counter negative, while the read path would ignore the result anyway. So the stored counter is explicitly **advisory only**; the live count is authoritative, and after a switch both trainers' live counts are automatically correct (A's active-link query loses this student, B's gains it) with no counter bookkeeping. §4.4 reflects this: the switch branch writes no counter.

**Justification.** The current `linkStudent` merges/re-links without a guard (the inventory flags this as the one behavioral gap). Silent replacement is explicitly forbidden by the requirement. A confirm-to-switch flow is chosen over hard rejection because the requirement's own example UI is a switch dialog and because trainers legitimately change (a student moves gyms); rejecting outright would strand such users. Routing the decision through an explicit `confirmSwitch` round-trip keeps the server as the single authority (the first call surfaces the conflict, the second performs the move only when intent is explicit), which fits the existing "never trust the client" posture — the flag is intent, the transaction is authority. The move is transactional for the same reason the original link is: the link row and the profile mirror must stay consistent.

### 2.4 Migration of the legacy `dreamphysics` trainer → **in-place normalization; nothing deleted, no duplicate created; seed repurposed as a generic bootstrap**

**Decision.** A one-shot, idempotent migration script (`backend/src/scripts/migrateReferralCodes.ts`) brings any pre-existing data up to the dynamic model without disconnecting anyone:
- For every `trainers/{trainerId}` doc, ensure it carries `referralCode`, `referralCodeLower`, `active:true`, `referralCodeUpdatedAt`, `createdAt`. The legacy `dreamphysics` trainer keeps its code — it simply becomes an ordinary unique code owned by that trainer, with no special-casing anywhere in code.
- For every `referralCodes/{codeLower}` doc, backfill the fields the new model expects (`code` = original display case if derivable else the normalized value, `active:true`, `createdAt`, `updatedAt`) while preserving the existing `trainerId`.
- `trainerLinks/*` are **not touched** — students linked through the old `dreamphysics` flow stay linked because their link is keyed by `studentUid`→`trainerId`, independent of the code.
- `backend/src/scripts/seedTrainer.ts` is **repurposed** into a generic, no-arg-required dev bootstrap: it no longer hardcodes `dreamphysics`/`Dream Physics` and no longer requires a privileged `SEED_TRAINER_UID`. It becomes an optional helper that, given a UID (or newly created dev user), promotes it to trainer and assigns a **generated** code via the same generator production uses. The `seed:trainer` npm script is kept only as this generic helper (or removed if the team prefers trainer self-registration to be the sole path — see §8). No single trainer is privileged, and the literal `dreamphysics` string is removed from all production/config code.

**Migration bypasses the format/reserved validator; the reserved list governs only NEW claims.** The migration backfills `trainers/*` and `referralCodes/*` directly via the Admin SDK and **does not run** the `referralCodeSchema` format or reserved-word validator. Existing codes — including the legacy `dreamphysics` — are **grandfathered active** exactly as stored. This is required because `dreamphysics` is on the reserved list (§4.6) specifically so that, after the legacy owner changes away from it, **no other trainer can re-claim it**; if the migration ran the validator it would reject its own legacy data. The reserved/format rules therefore apply **only to new claims** — `setCustomReferralCode`, `checkAvailability`, and generated candidates — never to migration backfill. Reconciliation of the two paths for the legacy trainer: that trainer may freely `generateReferralCode` or `setCustomReferralCode` to move to a new code; doing so **deactivates** (does not delete) the `referralCodes/dreamphysics` doc, and the reserved-word guard then keeps `dreamphysics` permanently unclaimable by any other trainer. A legacy trainer re-saving/regenerating runs through the normal (validated) new-claim path, which is fine because the new code they pick is validated while the old `dreamphysics` doc is just deactivated. If a legacy trainer's existing active code happened to collide with the reserved list (only `dreamphysics` does today), it stays grandfathered active and is simply unclaimable by others — no migration-time rejection.

**Justification.** The migration is additive and idempotent, matching the existing seed scripts' conventions (`hasFirebaseCredentials()` guard, merge writes). Because `trainerLinks` are UID-keyed and never code-keyed, normalizing codes cannot disconnect students — this is the crux that makes a safe, non-destructive migration possible. Keeping (not deleting) the legacy trainer and its code avoids both data loss and the creation of a duplicate trainer; the legacy trainer becomes indistinguishable from any dynamically created one, which is exactly the "no special-case logic for one trainer" requirement. Legacy student accounts with no `role` still resolve to `student` via the unchanged `roleService` (no write-back), preserving the existing no-write-on-unchanged-reopen invariant.

### 2.5 Code generation algorithm → **random, non-sequential, from an unambiguous alphabet, with collision-retry against the index**

**Decision.** Generated codes use a fixed prefix drawn from a small set plus a random suffix from an unambiguous alphabet (uppercase letters + digits excluding visually ambiguous `0/O/1/I`), e.g. `FIT7K2P9`, `TRN8X3Q`, length 7–8. Generation draws random characters via Node `crypto.randomInt`. Uniqueness is guaranteed by the single-doc-per-code index, never by client logic.

**Outer candidate loop; each attempt is one all-reads-before-writes transaction.** A Firestore transaction cannot internally invent a fresh candidate and retry, so the retry is **outer** and the claim is **inner**:

```
for attempt in 1..N (N = 5):
    candidate   = generateCandidate()            // pure, outside any transaction
    codeLower   = normalize(candidate)
    result = runTransaction(tx => {
        // READS FIRST
        newDoc = tx.get(referralCodes/{codeLower})
        oldDoc = oldCodeLower ? tx.get(referralCodes/{oldCodeLower}) : null
        // DECIDE
        if (newDoc.exists && newDoc.trainerId != me) return COLLISION   // abort this attempt, no writes
        // WRITES (only on success)
        tx.set(referralCodes/{codeLower}, { trainerId: me, code: candidate, active:true, createdAt?, updatedAt })
        if (oldDoc && oldCodeLower != codeLower) tx.update(referralCodes/{oldCodeLower}, { active:false, updatedAt })
        tx.update(trainers/{me}, { referralCode: candidate, referralCodeLower: codeLower, referralCodeUpdatedAt })
        return CLAIMED
    })
    if (result == CLAIMED) return candidate
// all N attempts collided:
throw ServiceUnavailableError(503)
```

Key guarantees this pins down (addressing the review's correctness concerns):
- **All reads precede all writes** inside every transaction (Firestore requirement). The new-code doc and the old-code doc are both read up front.
- **Claim + deactivate-old + update-trainer happen in the *same* successful transaction.** There is never a window with two active codes or zero active codes: the old code is only deactivated in the transaction that also activates the new one, and that transaction is atomic.
- **A collision aborts that attempt with zero writes** (we return before any `tx.set`), so the old code is **not** deactivated on a failed attempt — no double-deactivation and no half-applied state. The deactivation only runs in the attempt that actually claims the new code.
- The loop re-generates a **fresh** candidate per attempt; `503` is returned only if all N attempts collide (astronomically unlikely).

This same outer-loop/inner-transaction shape is used by `generateReferralCode` (regenerate: `oldCodeLower` is the trainer's current code) and by `ensureTrainerAccount`'s first-code generation (`oldCodeLower` is null, so the old-doc read and deactivate are skipped). `setCustomReferralCode` uses the **inner transaction once** (no outer loop — the candidate is the trainer's chosen code; a collision is a terminal `409`, not a retry).

**Justification.** `crypto.randomInt` is non-sequential and unpredictable, satisfying "do NOT use sequential predictable IDs." The restricted alphabet makes codes easy to type and read aloud (no `0`/`O` confusion). The transactional claim reuses the exact uniqueness guarantee the system already relies on (`referralCodes/{codeLower}` existence) rather than inventing a parallel mechanism. The outer loop with an all-reads-first inner transaction keeps a collision (vanishingly rare at this scale) a non-event while making the atomicity boundary explicit.

---

## 3. Data model changes (evolve existing documents — do not duplicate)

All changes are additive; existing documents remain valid, and the trainer record stays authoritative for ownership (`referralCodes` is a lookup index only).

### 3.1 `trainers/{trainerId}` (extend)
Already has `trainerId`, `name`, `photoUrl`, `referralCode`, `referralCodeLower`, `totalStudents`, `status`, `createdAt`, `updatedAt`. **Add**:
- `userId: string` — explicit mirror of the owning Firebase UID (equals `trainerId`; added because the target model names it `userId` and it documents intent; `trainerId` remains the doc id).
- `referralCodeUpdatedAt: timestamp | null` — set whenever the code is created/changed.
- `active: boolean` — trainer-account active flag the target model names, **a derived mirror only**. The existing `status: 'active' | 'inactive'` field is the **single source of truth** for whether a trainer account is active; `active` is written as `status === 'active'` whenever `status` changes and is never read for a decision. The §4.4 connect-path "trainer is active" check (step 2) reads **`status === 'active'`**, not `active`. The migration backfills `active = (status === 'active')`. `active` exists purely because the target data model names it; keeping `status` authoritative avoids the two-fields-drift hazard — there is exactly one field that decisions read.

### 3.2 `referralCodes/{codeLower}` (extend — still a lookup index, NOT source of truth)
Today stores only `{ trainerId }`. **Add** `code: string` (display form), `active: boolean`, `createdAt: timestamp`, `updatedAt: timestamp`. Doc id remains the normalized (lowercased, trimmed) code, preserving the O(1) case-insensitive lookup and single-doc-per-code uniqueness. The connect path treats a missing doc **or** `active:false` as "code not valid." Ownership is always confirmed against the `trainers/{trainerId}` record, never inferred from this collection alone.

### 3.3 `trainerLinks/{studentUid}` (unchanged shape)
Keyed by `studentUid`, carries `trainerId`, `status`, `createdAt`, `updatedAt`. **No change** — the relationship is already keyed by IDs, never by code or email. The switch flow (§2.3) updates `trainerId` in place within a transaction; it never changes the doc id.

### 3.4 `users/{uid}` / `users/{uid}/profile/data` (unchanged)
`role?: 'student' | 'trainer'` and the profile mirror `trainerId` / `shareProgressWithTrainer` already exist from the prior task. Trainer self-registration writes `users/{uid}.role='trainer'`; nothing else changes.

### 3.5 `firestore.indexes.json` (no new index required)
The only cross-trainer query is still `trainerLinks where trainerId==, status=='active'`, covered by the existing composite index `trainerLinks (trainerId ASC, status ASC, updatedAt DESC)`. Code lookups are single-document `get`s by id and need no index. **No index change is needed**; this is called out explicitly so the implementer does not add a redundant one.

---

## 4. Backend design

New/changed files on the `/api/trainer`, `/api/student`, and `/api/auth` slices, following existing conventions exactly (`routes → controllers → services → repositories`, Zod validators, `ok()` envelope, `utils/errors.ts` subclasses mapped by `errorHandler.ts`).

### 4.1 Referral-code generator (new util)
`backend/src/utils/referralCode.ts` — `generateCandidate(): string` (prefix + random suffix per §2.5) and `normalize(code): string` (`trim().toLowerCase()`). Pure and unit-testable (inject the random source for deterministic tests). The uniqueness **claim** lives in the repository/service transaction, not here.

### 4.2 Trainer repository (extend `trainerRepository.ts`)
Reuse `get`, `upsert`, `findTrainerIdByReferralCode`, `countActiveStudents`, `incrementTotalStudents`. The existing `setReferralCode(codeLower, trainerId)` (a bare `{ trainerId }` upsert) is **superseded** by `claimReferralCode` + `setActiveReferralCodeOnTrainer` below and will be **removed** (or kept only as a thin wrapper that calls `claimReferralCode`); it is not used by any new path, because it writes neither the `code`/`active`/timestamp fields the new model needs nor performs the ownership check. **Add**:
- `getByReferralCodeDoc(codeLower): { trainerId, active } | null` — reads the full `referralCodes/{codeLower}` doc (so the connect path can check `active`, not just existence).
- `claimReferralCode(codeLower, displayCode, trainerId, tx)` — inside a transaction (reads already done per §2.5), applies the **shared claimability predicate** (below) and, if claimable by `trainerId`, writes `{ trainerId, code: displayCode, active:true, createdAt (first write only), updatedAt }`. Used by both generate and custom-set.
- `deactivateReferralCode(codeLower, tx)` — set `active:false, updatedAt` on the old code doc during a change.
- `setActiveReferralCodeOnTrainer(trainerId, displayCode, codeLower, tx)` — update `trainers/{trainerId}` `referralCode`, `referralCodeLower`, `referralCodeUpdatedAt`.

**Shared claimability / availability predicate (single definition, used by both claim and availability — resolves the prior ambiguity).** Given a target `codeLower` and the requesting trainer `T`, read the `referralCodes/{codeLower}` doc. Then:

> **Claimable by `T`** iff the doc (a) **does not exist**, or (b) **exists and `trainerId === T`** (re-claim of your own code — active *or* inactive; a no-op if already yours and active).
> **Unavailable** iff the doc **exists with `trainerId !== T`**, **regardless of the doc's `active` flag**. A deactivated code owned by trainer A is therefore **never** handed to trainer B — this preserves A's retained audit doc (§2.2) and prevents B from overwriting it.

`claimReferralCode` enforces exactly this: it writes only when claimable-by-`T`; otherwise it returns a `taken` signal (→ `409` for custom-set, → next candidate for generate). `checkAvailability` returns `available:false, reason:'taken'` for the unavailable case. The predicate is defined once here and referenced verbatim by §4.3 and the §4.7 table, so claim and availability can never disagree.

### 4.3 Trainer referral-code service (extend `trainerService.ts`)
- `ensureTrainerAccount(uid, { name, email, photoUrl })` — called by trainer registration: upsert `users/{uid}.role='trainer'` and `trainers/{uid}` (createdAt on first write, `active:true`, `userId:uid`), then **generate and claim** a unique code (bounded retry, §2.5) if the trainer has none yet. Idempotent: re-running does not create a second trainer or a second code.
- `getReferralCode(trainerId)` — return the trainer's current `{ code, active, referralCodeUpdatedAt }`.
- `generateReferralCode(trainerId)` — runs the **outer candidate loop with the inner all-reads-first transaction defined in §2.5** (claim new + deactivate old + update trainer atomically per attempt; `503` only if all N attempts collide). Returns the new code. (Powers "regenerate".)
- `setCustomReferralCode(trainerId, desiredCode)` — validate format + reserved words (validator §4.6), normalize, then run the **inner transaction from §2.5 exactly once** (no outer loop): read new-code doc + old-code doc, apply the shared claimability predicate (§4.2) and `409 ConflictError` if the code is owned by another trainer, else write new-code active + deactivate old + update trainer, all atomically. (Powers custom code "Save".)
- `checkAvailability(desiredCode)` — validate format + reserved words first (invalid format → `reason:'invalid'`; reserved → `reason:'reserved'`), normalize, then apply the **shared claimability predicate** from §4.2: `available:true` iff the code doc does not exist **or** is owned by the requesting trainer; `available:false, reason:'taken'` iff the doc exists and is **owned by a different trainer, regardless of `active`**. Read-only, no write.

### 4.4 Connect/link service (extend `linkStudent` with the switch guard, §2.3)
Signature becomes `linkStudent(studentUid, referralCode, opts?: { confirmSwitch?: boolean })`. Flow:
1. Normalize code; read `referralCodes/{codeLower}` via `getByReferralCodeDoc`. If missing or `active:false` → `{ linked:false, reason:'invalid_code' }` (HTTP 200, soft — unchanged behavior for typos).
2. Resolve the owning `trainerId`; load the trainer and **verify the resolved account is actually a trainer** (`roleService.resolveRole(trainerId) === 'trainer'` and `trainers/{trainerId}` exists and `active`); if not → `{ linked:false, reason:'invalid_code' }` (never link to a non-trainer).
3. A `trainer`-role **student caller** cannot link as a student → `409 ConflictError` (unchanged).
4. Read `trainerLinks/{studentUid}`:
   - No active link → create it via the **existing inline transactional `tx.set(linkRef, …)` in `linkStudent`** (there is no `trainerLinkRepository.create`; the link doc is built inline today): link row, mirror `profile.trainerId`, increment the trainer's stored counter once (unchanged create-path behavior). Return `{ linked:true, trainerId, trainerName }`.
   - Active link to the **same** trainer → idempotent no-op `{ linked:true, trainerId, trainerName, alreadyLinked:true }` (no double count).
   - Active link to a **different** trainer and `confirmSwitch !== true` → `{ linked:false, reason:'already_linked', currentTrainer, requestedTrainer }` (HTTP 200). **No write.** `currentTrainer` is built by loading trainer A's doc; if A was deleted/deactivated the load yields `{ trainerId, name: null }` — this is tolerated (the client dialog degrades to "your current trainer", §5.4), never an error.
   - Active link to a **different** trainer and `confirmSwitch === true` → transactional switch that updates **only** the link row (`trainerId` → B, `updatedAt`) and re-mirrors `profile.trainerId`. **No stored-counter mutation** (per §2.3: the dashboard uses the live `countActiveStudents`, which is automatically correct after the move; introducing a decrement is the system's first-ever decrement and could go negative on a drifted counter). Return `{ linked:true, switched:true, trainerId, trainerName }`.

This keeps the student connect path backward-compatible (empty/valid/invalid behave exactly as before) and adds only the already-linked branch.

### 4.5 Endpoints (routes + controllers)
On the existing `/api/trainer` slice (behind `authenticate` + `requireTrainer`):

| Method & path | Purpose | Service |
|---|---|---|
| `GET /api/trainer/referral-code` | Fetch the trainer's current code | `getReferralCode` |
| `POST /api/trainer/referral-code` | Generate a new unique code (regenerate) | `generateReferralCode` |
| `PATCH /api/trainer/referral-code` | Set a custom code | `setCustomReferralCode` |
| `GET /api/trainer/referral-code/availability?code=` | Check-availability affordance | `checkAvailability` |

On `/api/auth` (authenticated):

| Method & path | Purpose | Service |
|---|---|---|
| `POST /api/auth/register-trainer` | Promote the just-created Firebase user to a trainer (role + trainer doc + generated code) | `ensureTrainerAccount` |
| `POST /api/auth/link-trainer` (evolve) | Connect a student by code; now resolves `active` codes dynamically and honors the switch guard (body gains optional `confirmSwitch:boolean`) | `linkStudent` |

On `/api/student` (authenticated): `POST /api/student/connect-trainer` (connect-after-registration from Profile — thin alias delegating to `linkStudent`, same body/semantics as `link-trainer`, so the student app has a role-appropriate route) and the existing `GET /api/student/trainer`. Both `link-trainer` and `connect-trainer` share one service method so behavior cannot drift.

`register-trainer` sets the role **server-side** from the verified token's UID; the client never asserts a role. Role is still resolved on every `/api/auth/verify` by `roleService`, so the router sees `trainer` on the next verify and routes to the trainer shell.

### 4.6 Validation (Zod, backend — never client-only)
- **Connect code** (`linkTrainerSchema`, reuse + extend): `referralCode` required, `trim().toLowerCase()`, min 3 / max 40; add optional `confirmSwitch: z.boolean().optional()`. The service additionally checks existence + `active` + resolves the exact trainer + verifies the resolved account is a trainer (§4.4). Reused verbatim by `connect-trainer`.
- **Custom/generated code** (new `referralCodeSchema`): `code` required, `trim()`, length **4–20**, regex `^[A-Za-z0-9]+$` (letters + digits only, no spaces/symbols), not a reserved word (case-insensitive match against a small list, e.g. `admin`, `trainer`, `fittrack`, `support`, `null`, `undefined`, `dreamphysics` reserved so it can't be re-claimed by someone else post-migration). Normalization to lowercase happens for the index key; the display form preserves the trainer's casing. On failure → `422` via `validate` middleware (format) or `409 ConflictError` (taken by another trainer) / `400` (reserved).
- **Availability query** (`availabilityQuerySchema`): `code` required, same format rules; returns `{ available, reason }` (no write).
- **Trainer registration** (`registerTrainerSchema`): `name` required (1–80), `email`/`photoUrl` optional and taken from the verified token where possible; **no referral-code field** (choosing TRAINER must not require a code).

Each external input states required/optional, type, limits, and failure behavior above. Invariants: **code uniqueness** is owned by the repository transaction (`claimReferralCode`) because it is the only layer that can atomically check-and-write the index; **role** is owned by the auth/middleware layer (`roleService` + `requireTrainer`) because it must be server-trusted; **link consistency / counters** are owned by the service transaction because they span multiple docs.

### 4.7 Error handling (concrete, per failing operation)
Uses existing `utils/errors.ts` + `errorHandler.ts`; success via `ok()`.

| Operation / condition | Recoverable? | Caller receives | Logged? |
|---|---|---|---|
| `register-trainer`, missing/invalid token | yes (re-auth) | `401 UnauthorizedError` | `warn` (code only) |
| `register-trainer`, already a trainer | yes (idempotent) | `200` with existing trainer+code (no duplicate) | `info` |
| `generate`, generated candidate collides (owned by another trainer) | yes (outer loop picks a fresh candidate, §2.5) | transparent; `503 ServiceUnavailableError` only if all N attempts collide (never expected) | `warn` on each retry, `error` if exhausted |
| custom-set, code owned by another trainer (per §4.2 predicate, regardless of `active`) | yes (pick another) | `409 ConflictError` "That referral code is taken." | `info` |
| custom-set/availability, format invalid | yes | `422` with field message | none |
| custom-set/availability, reserved word | yes | `400 BadRequestError` "That code is reserved." | `info` |
| connect, code unknown or inactive | yes | `200 { linked:false, reason:'invalid_code' }` | `info` |
| connect, resolved account is not a trainer | yes | `200 { linked:false, reason:'invalid_code' }` (never leak internal state) | `warn` |
| connect, already linked to a different trainer, no confirm | yes | `200 { linked:false, reason:'already_linked', currentTrainer, requestedTrainer }` | `info` |
| connect switch, confirmed | n/a | `200 { linked:true, switched:true, ... }` | `info` |
| connect, trainer account tries to link as student | no | `409 ConflictError` | `warn` |
| any trainer route, role≠trainer | no | `403 ForbiddenError` | `warn` with uid |
| Firestore/Admin unavailable | maybe | `503 ServiceUnavailableError` | `error` |
| unexpected | no | `500` generic | `error` with stack |

The connect path deliberately keeps "unknown code" and "resolved account not a trainer" indistinguishable to the caller (both `invalid_code`) so the API never reveals internal data shapes; the distinction is only in logs.

### 4.8 Security (preserve existing authorization; referral codes grant no access)
- Referral codes are **data for lookup**, never credentials: possessing a code lets a student *request* a link, but every `/api/trainer/*` call is still gated by `authenticate` + `requireTrainer` + `assertTrainerOwnsStudent`. A student knowing trainer A's code gains no read access to A's students. This is stated explicitly because the requirement stresses "referral codes NEVER grant API access."
- `trainers`, `referralCodes`, `trainerLinks` remain **client-deny** in `firestore.rules` (`allow read, write: if false`), fail-closed; all access is through the Admin SDK on the backend. The new code-management endpoints write these collections only via the backend. **No rule change is required** for the dynamic model, and none that would open public reads is permitted. The migration/bootstrap scripts use the Admin SDK (bypass rules) exactly like the existing seed scripts.
- Existing per-student authorization is unchanged: trainer A reads only A's students (`assertTrainerOwnsStudent` → `404` for unowned, not `403`, so another trainer's roster is not even revealed to exist); students read only their own data and their trainer's public subset via `GET /api/student/trainer`.

---

## 5. Frontend design

New/changed files on the auth and trainer slices; the student `HomeShell`, trainer `TrainerShell`, and Midnight Energy theme are preserved.

### 5.1 Account-type selector at registration (`register_screen.dart`)
Add an **account-type selector above the form fields**, before any account is created. UI: a **segmented control / two selectable cards** `[ USER ] [ TRAINER ]` built from existing `AppColors`/`AppShadows`/`AppSpacing` so it matches the Midnight Energy auth UI; the selected card is visually obvious (filled gradient vs outline). State is a local `_AccountType _type = _AccountType.user`.
- **USER selected** (default): the form is exactly today's — Name, Email, Password, Confirm Password, and the existing optional "Trainer referral code" field with hint `Enter trainer referral code (optional)`. The existing register→`linkTrainerControllerProvider.link(code)` flow is unchanged.
- **TRAINER selected**: hide the referral-code field entirely (choosing trainer must not require a code). After a successful `register()`, the client calls a new `authRepository.registerTrainer(name)` (hitting `POST /api/auth/register-trainer`) instead of `linkTrainer`. The role becomes `trainer` server-side; on the next `verify` the router routes to the trainer shell. Field set: Name, Email, Password, Confirm Password, and (if the profile architecture later supports it) a profile photo — photo is **optional and deferred** because trainer photo upload reuses the existing Storage pipeline and is not required for a working trainer account; the trainer record stores `photoUrl:null` until set.

Because the register screen is torn down the moment registration succeeds (the router redirects to verify-email), the trainer-promotion call is owned by a root-container controller analogous to the existing `LinkTrainerController` — a `RegisterTrainerController` on the root `ProviderContainer` so the request completes after the screen is disposed. (Reuses the exact pattern already proven for `linkTrainerControllerProvider`.)

### 5.2 Trainer referral-code management in Profile (`trainer_profile_screen.dart` → `_ReferralCard`)
Extend the existing `_ReferralCard` (currently display + copy) with:
- **Copy** (keep as-is).
- **Change / Generate New** button → opens a bottom sheet with two options: "Generate a new code" (calls `POST /api/trainer/referral-code`) and "Choose a custom code" (text field + **Check Availability** button hitting `GET /api/trainer/referral-code/availability?code=`, live `✓ Available` / `✗ Taken` / format error, then **Save** → `PATCH /api/trainer/referral-code`).
- On success, invalidate `trainerProfileProvider` so the card shows the new code; show a SnackBar. The card reads the code dynamically from `TrainerProfile.referralCode` (already wired) — no hardcoded code anywhere on the client.

A trainer who skipped code setup still has a generated code (registration always generates one, §4.3), so the card is never empty; the "Change" affordance covers the "create it later" requirement.

### 5.3 Connect-after-registration from student Profile (`profile_screen.dart`)
The student Profile "My Trainer" area gains two states driven by `GET /api/student/trainer` (`MyTrainer`):
- **No trainer** (`MyTrainer.none`) → a **"Connect to Trainer"** affordance that opens a sheet with a code field → `POST /api/student/connect-trainer`. On `invalid_code` show "Invalid trainer referral code"; on `already_linked` show the **switch confirmation** dialog (§5.4); on success show "Connected to <trainer>" and invalidate the trainer provider.
- **Connected** → the existing "My Trainer" card (name, photo, status) plus a referral display, unchanged.

### 5.4 Switch-trainer confirmation (client half of §2.3)
When connect returns `{ reason:'already_linked', currentTrainer, requestedTrainer }`, show a dialog: "You're connected to **{currentTrainer.name}**. Switch to **{requestedTrainer.name}**?" `[ Cancel ] [ Switch Trainer ]`. If `currentTrainer.name` is null (trainer A was deleted/deactivated, §4.4), the copy **degrades to "your current trainer"** rather than rendering an empty name. **Switch** re-calls `connect-trainer` with `confirmSwitch:true`; on success invalidate `myTrainer` + any trainer-dependent providers. Cancel leaves the existing link intact. No local relationship state is mutated — the server is the authority.

### 5.5 Data layer (extend `trainer_repository.dart` + `auth_repository.dart`)
- `trainer_repository.dart` gains `getReferralCode()`, `generateReferralCode()`, `setCustomReferralCode(code)`, `checkAvailability(code)` and `connectTrainer(code, {confirmSwitch})`, each returning `Result<T>` and parsing the `{ success, data }` envelope like the existing methods.
- `auth_repository.dart` gains `registerTrainer(name)` → `POST /api/auth/register-trainer`. `linkTrainer` gains an optional `confirmSwitch` param (defaulting false) so the existing registration link path is unchanged.
- New small models: `ReferralCodeInfo { code, active, updatedAt }`, `AvailabilityResult { available, reason }`, and a `ConnectResult` superset of `LinkTrainerResult` carrying `reason:'already_linked'` + `currentTrainer`/`requestedTrainer`. `TrainerProfile`/`MyTrainer` are reused.

### 5.6 Role-based routing (unchanged in spirit)
`app_router.dart` already routes `role=='trainer'` → `TrainerShell` **before** the onboarding gate and everyone else through the student flow. **No router change** is required: a freshly promoted trainer gets `role:'trainer'` on the next `verify` and lands on the trainer shell; students keep the existing `HomeShell` + onboarding. This is stated so the implementer does not re-touch the redirect.

### 5.7 Client edge cases
- Trainer registration call fails after the Firebase user is created → the account exists as a (role-less → student) user; the client surfaces a retryable "Couldn't finish trainer setup" and can re-call `register-trainer` (idempotent). The user is never stranded.
- Availability check races a save (someone claims the code between check and Save) → the `PATCH` returns `409`; the UI shows "That code was just taken, try another."
- Generated-code regenerate while students are connected → students stay connected (links untouched, §2.2); the UI notes "Your old code no longer works for new students."
- Connect with an empty code from Profile → no call made (same as registration's empty-field behavior).

---

## 6. Migration & bootstrap (operational)

`backend/src/scripts/migrateReferralCodes.ts` (new, idempotent, Admin SDK, `hasFirebaseCredentials()` guard): backfills the new fields on all `trainers/*` and `referralCodes/*` docs (§2.4), touches no `trainerLinks`. Safe to run multiple times. Added as an npm script `migrate:referral-codes`.

`backend/src/scripts/seedTrainer.ts` (repurpose): drop the `dreamphysics`/`Dream Physics`/`SEED_TRAINER_UID`-privileged constants; become a generic dev helper that promotes a given/created UID to trainer with a **generated** code via the production generator. Not a privileged bootstrap; may be removed if the team prefers self-registration only. Update `trainerValidators.ts` doc comment that references "the only seeded code today is 'dreamphysics'."

Backward compatibility: existing users and existing trainer-student relationships keep working unchanged; `dreamphysics`-linked students are not disconnected (their links are UID-keyed). Legacy accounts with no role still resolve to `student` with no write-back.

---

## 7. Testability

**Backend unit (vitest, in-memory Firestore like `userRepository.test.ts`):**
- `referralCode` generator: non-sequential, alphabet excludes `0/O/1/I`, length bounds, deterministic with injected RNG.
- `claimReferralCode`: first claim succeeds; second claim of same code by a different trainer fails; same trainer re-claim is a no-op.
- `generateReferralCode` / `setCustomReferralCode`: new code active, old code deactivated, trainer record updated, **`trainerLinks` untouched** (assert link docs unchanged before/after).
- `checkAvailability`: available / taken-by-other / reserved / invalid-format.
- `linkStudent` switch guard: no link → links; same trainer → idempotent no double-count; different trainer no confirm → `already_linked` and **no write**; different trainer with `confirmSwitch` → link `trainerId` updated to B + profile re-mirrored, and **neither trainer's stored `totalStudents` is written** (assert the stored counters are unchanged by the switch, and that `getProfile`'s live `countActiveStudents` reports A−1 / B+1); `already_linked` current-trainer name null when A's doc is missing.
- `ensureTrainerAccount` idempotency (keep per review #9): creates role+trainer+code once; **re-run creates no duplicate trainer and no second code** (same `referralCode` before/after a second call), so a failed-then-retried trainer registration never doubles up.
- Generated-code transaction shape: a forced single collision (first candidate's doc pre-exists owned by another trainer) retries to a fresh candidate and still ends with exactly one active code + the old one deactivated; a failed attempt leaves the old code's `active` **unchanged** (no deactivation on an aborted attempt).
- Validators: `referralCodeSchema` format/reserved; `linkTrainerSchema` with `confirmSwitch`.

**Backend integration (supertest-style over the app):** `register-trainer` (401 no token, 200 promotes, idempotent re-call); referral-code endpoints 401/403 (student)/200 (trainer); connect-trainer valid/invalid/already-linked/confirmed-switch; the full existing `/api/trainer/*` authorization matrix still green.

**Firestore rules (emulator, integration-only):** `trainers`/`referralCodes`/`trainerLinks` remain non-client-readable/writable after the change (no regression).

**Flutter widget tests:** account-type selector toggles field visibility (USER shows referral field, TRAINER hides it); trainer-profile change/generate flow updates the displayed code via overridden providers; student connect-from-profile shows invalid/already-linked/switch outcomes; register-as-trainer routes to the trainer shell (role-driven).

**Hard-to-test seams:** the code generator's randomness is isolated behind an injectable RNG so services are deterministic in tests; real Firestore transactions are exercised via the in-memory mock. Migration/bootstrap scripts require real credentials and are run manually, not in CI (consistent with the existing seed-script policy).

Verification gate — backend (`backend/`): `npm run typecheck`, `npm run lint`, `npm test`. Frontend (`frontend/`): `flutter analyze`, `flutter test`, `flutter build apk --debug`.

---

## 8. Open items / assumptions

- **`seed:trainer` fate.** Assumed repurposed into a generic dev helper (not removed) so local dev can create a trainer without the full UI; the team may delete it once self-registration is the only path. Either choice satisfies "remove the single-hardcoded-trainer seed."
- **Trainer profile photo at registration.** Assumed optional/deferred (trainer record stores `photoUrl:null` until set) because photo upload reuses the existing Storage pipeline and is not required for a working trainer account. If the review wants photo-at-registration, it slots into the existing profile-photo upload flow with no model change.
- **`firebase/firestore.rules` is unversioned.** No rule change is required by this design; if the implementer touches it for any reason, flag it for user review.
- **Reserved-word list** is a small fixed set in code; `dreamphysics` is reserved post-migration so it can't be re-claimed by a different trainer.

---

## 9. Out of scope / non-regression

**Out of scope:** trainer-initiated writes to student data; messaging/chat; multiple active trainers per student (one active trainer is enforced); trainer teams/orgs; payment/billing; per-photo (vs per-student) sharing granularity; redesigning any existing student screen beyond the additive selector, referral management, and connect-from-profile affordances.

**Must not regress:**
- The nutrition Add-Food fix: `nutritionService.getDay` preserves `mealId`/`mealName`; `addFood` honors client-supplied `clientId` idempotency and never persists `clientId`. No file this design touches overlaps the nutrition read/write path.
- Existing trainer-student authorization (trainer reads only own students, 404 for unowned; students read only their own data + trainer public subset).
- Role-based routing, student onboarding, all student features, and the Midnight Energy theme.
- `firestore.rules` stays fail-closed with no public student reads; referral codes grant no API access.
- Existing `dreamphysics`-linked students remain connected after migration.

---

## 10. Responses to design-review findings

Verdict was CHANGES_REQUESTED (0 HIGH, 4 MEDIUM, 5 NIT). Every finding is addressed below; the choices align with the original requirements (dynamic multi-trainer, no silent overwrite, non-destructive migration, server-trusted role, no API access from codes, no nutrition regression).

- **#1 (MEDIUM) — `claimReferralCode` predicate ambiguous / conflicts with availability.** Resolved. §4.2 now defines **one shared claimability/availability predicate** and both the claim path and `checkAvailability` reference it verbatim: claimable iff the code doc does not exist OR is owned by the same trainer; unavailable iff it exists with a different `trainerId` **regardless of `active`** (so a deactivated code owned by A is never handed to B, preserving A's §2.2 audit doc). §4.3 `checkAvailability` wording and the §4.7 table row were aligned to "owned by a different trainer."

- **#2 (MEDIUM) — generated-code retry loop vs single-transaction semantics.** Resolved. §2.5 now pins the shape with pseudocode: an **outer candidate loop (≤5 attempts)**, each attempt a **single transaction with all reads before all writes** that claims the new code, deactivates the old code, and updates the trainer atomically; a collision aborts the attempt with **zero writes** (so no double-deactivation and never a two-active-or-zero-active window); `503` only on exhaustion. §4.3 states `generateReferralCode` uses this loop and `setCustomReferralCode` uses the inner transaction once (collision → terminal `409`, no retry).

- **#3 (MEDIUM) — reserved list blocks `dreamphysics` while migration keeps it live.** Resolved. §2.4 now states explicitly that the migration **backfills via the Admin SDK without running the format/reserved validator**, grandfathering existing codes (including `dreamphysics`) as active, and that the reserved list applies **only to new claims** (`setCustomReferralCode`, `checkAvailability`, generated candidates) — specifically so no *other* trainer can reclaim `dreamphysics` after the legacy owner changes away. It also notes a legacy trainer regenerating deactivates (never deletes) the `dreamphysics` doc, which the reserved guard then keeps permanently unclaimable by others.

- **#4 (MEDIUM) — switch-flow counter decrement can go negative / inconsistent.** Resolved by **dropping stored-counter maintenance from the switch path entirely** (the preferred option). §2.3 and §4.4 now state the switch transaction writes only the link row + profile mirror and touches **no** stored counter, because `getProfile` already derives the displayed count from the live `countActiveStudents` query (verified: it is the only reader and overwrites the stored field). The stored counter is declared advisory only; after a switch both trainers' live counts are automatically correct with no bookkeeping, so no decrement can go negative.

- **#5 (NIT) — `setReferralCode` not reconciled.** Resolved. §4.2 states `setReferralCode` is **superseded** by `claimReferralCode` + `setActiveReferralCodeOnTrainer` and will be removed (or kept only as a thin wrapper), since it writes neither the new fields nor the ownership check.

- **#6 (NIT) — "existing create path" overstates reuse.** Resolved. §4.4 now says the no-link branch creates the link via **"the existing inline transactional `tx.set(linkRef, …)` in `linkStudent`"** and notes there is no `trainerLinkRepository.create`.

- **#7 (NIT) — `active` vs `status` dual field invites drift.** Resolved. §3.1 names **`status === 'active'` the single source of truth**; `active` is a derived mirror (written as `status === 'active'`, never read for a decision). §4.4 step 2's trainer-active check reads `status === 'active'`; the migration backfills `active` from `status`.

- **#8 (NIT) — trainer-not-found on `already_linked` path unspecified.** Resolved. §4.4 states `currentTrainer.name` **may be null** when trainer A is deleted/deactivated (tolerated, not an error); §5.4 degrades the dialog copy to "your current trainer" when the name is null.

- **#9 (NIT) — register-trainer leaves a brief role-less window.** Acknowledged as already handled (retryable `RegisterTrainerController` + idempotent endpoint, §5.1/§5.7). Per the finding's request, §7 **retains the `ensureTrainerAccount` idempotency test** (re-run creates no duplicate trainer and no second code). Not a blocker.
