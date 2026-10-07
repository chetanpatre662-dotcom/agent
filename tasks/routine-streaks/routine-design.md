# FitTrack — Daily Routine / Goals + Progress Calendar + Strict Dual-Streak — Technical Design

Status: **revision 3** — revision 2 resolved the first review's HIGH/MEDIUM/NIT findings; revision 3 resolves the one MEDIUM finding in the latest `design-review.json` / `design-review.md` (the §10 validation-failure status codes were 400 but the real pipeline returns 422 for `ZodError`). See the "Review response" sections at the end for the point-by-point mapping.
Scope: `frontend` (Flutter/Dart, Riverpod + GoRouter + Dio), `backend` (TypeScript/Express + Firestore Admin SDK).
Grounded in `routine-audit.md`. The central decision from the audit is honored: **extend the existing `routine` feature in place; do not build a sibling "goals" concept.** The existing `Routine`/`routineCompletions`/`RoutineDay` model already is the generic daily-goal concept.

---

## 1. Overview

The existing routine feature already models generic daily tasks (`users/{uid}/routines`) with a separate per-day completion collection (`users/{uid}/routineCompletions`), a "today" aggregate (`RoutineDay`), reminder scheduling, and a dashboard card. This design extends that foundation additively to deliver: the full field set the spec asks for (priority, start/end time, `once`+date scheduling, `isRequiredForStreak`, `linkedWorkoutId`), a monthly progress calendar with per-day states, a day-detail view, two strictly independent activity-based streaks (Routine + Workout), a user-chosen dashboard streak display, and per-task reminders through the existing notification system.

Two behavior corrections are mandatory and are treated as first-class design items:
1. **Deleting a routine must stop purging completion history.** The current `routineService.delete` → `deleteCompletionsForRoutine` is a spec violation. We switch to soft-delete (archive).
2. **The dashboard "streak" must be an activity-based streak chosen by the user**, not the morning-game streak that is currently wired into the banner. The morning-game streak logic (`streakCalc.computeStreak` fed from `gameHistory`) is a preserved prior fix and stays — it simply stops being the dashboard's generic banner source.

**Source-of-truth decision (locked): the BACKEND computes all streaks and monthly aggregates.** Rationale: the `routineCompletions` and `workouts` records already live server-side; the reusable, unit-tested `computeStreak` already runs server-side; computing on the client would require downloading full history on every device and would duplicate the "which days qualify" rule in Dart. A single backend computation keeps one authoritative answer across Home/Routine/Workout/Profile and keeps the strict-reset rule testable with Vitest. The Flutter `StreakService` is therefore a thin **client façade** over backend endpoints (not a reimplementation) — it is still the single consumption point so no screen computes streaks locally.

**Technology stack (locked once approved):** TypeScript/Express + Firebase Admin Firestore + Zod validators + Vitest (backend); Flutter + Riverpod (`StateNotifierProvider`/`FutureProvider`) + GoRouter + Dio `ApiClient` + `flutter_local_notifications` + `shared_preferences` + `flutter_test` (frontend). No new packages, no dependency upgrades, no APK build (hard constraints from audit §13).

---

## 2. Data models

### 2.1 Routine (extended, backward compatible)

Backend wire shape (persisted at `users/{uid}/routines/{routineId}`). New fields are additive; existing docs without them read with defaults so **no data migration of routine definitions is required**.

| Field | Type | Notes |
| --- | --- | --- |
| `id` | string | Firestore doc id. |
| `userId` | string | **Structural** — the doc lives under `users/{uid}`. We do **not** store a redundant `userId` field inside the doc (matches existing repos); ownership is the path. The task's model lists `userId`; it is represented by the collection path `uid`, and the API never trusts a client-supplied userId. |
| `name` | string(1..120) | Task/goal title (existing field; spec "title"). |
| `description` | string(≤500)? | Optional; stored as `null` when absent (existing convention). |
| `category` | enum | See §2.2. |
| `time` | `HH:mm` | Treated as **start time** (existing field reused). |
| `endTime` | `HH:mm`? | **New**, optional end time. |
| `repeat` | `once \| daily \| selected` | **New** scheduling mode. Absent ⇒ inferred `daily`/`selected` from `repeatDays` (back-compat). |
| `date` | `YYYY-MM-DD`? | **New**, required only when `repeat == once`. |
| `repeatDays` | `number[]` (0=Sun..6=Sat) | Existing; used when `repeat == selected`. `daily` ⇒ all 7 / empty. |
| `priority` | `low \| medium \| high` | **New**, default `medium`. |
| `isRequiredForStreak` | boolean | **New**, default `true`. Core to Routine Streak. |
| `linkedWorkoutId` | string? | **New**, optional. When set, completing that specific workout auto-satisfies this routine. |
| `enabled` | boolean | Existing; reused as **pause/resume** (paused = `enabled:false`). Already excludes from "today" view and reminder scheduling. |
| `notificationEnabled` | boolean | Existing reminder toggle. |
| `isActive` | boolean | **New**, default `true`. `false` = soft-deleted/archived (see §4.3). Distinct from `enabled` (pause) so a paused routine and a deleted routine are different states. |
| `createdAt` | server timestamp | **Write-once.** Set only on create; never rewritten (see §2.4). |
| `updatedAt` | server timestamp | Set only on a real change (see §2.4). |
| `status` | derived | Not persisted on the routine. Only present on the `RoutineDay`/day-detail DTO, resolved from completions. |

> Field mapping note: the task's model uses `title`/`startTime`/`repeatType`/`repeatDays`. We keep the existing persisted names `name`/`time`/`repeat`/`repeatDays` to avoid a rename migration of live docs and to keep the frontend `Routine` model stable. The design satisfies the semantics the task requires; the only naming divergence is `title→name`, `startTime→time`, `repeatType→repeat`. This is an explicit, intentional choice to extend rather than rewrite.

### 2.2 Category enum (extended)

Audit decision adopted: **extend both enums additively** rather than lossy-map the new labels. Append to `backend/src/models/domain.ts::ROUTINE_CATEGORIES` and the frontend `RoutineCategory` (kept in sync per the `domain.ts` header mandate):

Existing (kept): `wake_up, breakfast, lunch, dinner, snack, workout, water, sleep, study, work, custom`.
Add: `morning, health, personal, other`.

The spec's six primary categories map as: Morning→`morning`, Study→`study`, Workout→`workout`, Health→`health`, Personal→`personal`, Other→`other`. The legacy values remain valid for existing docs and the lifestyle meal/water routines. Frontend `RoutineCategory.fromWire` already falls back to `custom` on unknown values, so adding wire values is backward/forward compatible. Add `label`, `wire`, and an icon mapping for each new value (icons: morning→`wb_twighlight`/`wb_sunny_outlined`, health→`favorite_outline`, personal→`person_outline`, other→`event_note_outlined`).

### 2.3 RoutineCompletion (separate collection — unchanged location, extended fields)

Persisted at `users/{uid}/routineCompletions/{dateKey__routineId}` (existing deterministic doc id). Completion is **never** a bool on the routine; recurring routines keep one record per day so history survives definition edits/deletes.

| Field | Type | Notes |
| --- | --- | --- |
| `id` | string | `${dateKey}__${routineId}` (deterministic ⇒ idempotent, no duplicate on retry/offline sync). |
| `routineId` | string | Existing. |
| `dateKey` | `YYYY-MM-DD` | Existing; local day. Indexed for range queries. |
| `status` | `pending \| completed \| skipped` | Existing. |
| `completedAt` | server timestamp? | **New.** Set when `status→completed`; cleared (`null`) when moved back to `pending`. Satisfies "record completion timestamp" (spec item 3). |
| `userId` | — | Structural (collection path), not stored. |
| `createdAt` | server timestamp | **New, write-once** — first time this completion doc is written. |
| `updatedAt` | server timestamp | Existing; set only when `status`/`completedAt` actually change (idempotent write, §2.4). |

A `RoutineCompletion` document is self-sufficient for calendar/streak history (`dateKey`, `routineId`, `status`) and survives routine soft-delete.

### 2.4 Write-once `createdAt` / idempotent `updatedAt` (reuse userRepository pattern)

`userRepository.ensureAccount`/`addFcmToken` already demonstrate the pattern: read current doc, write only when a tracked field changed, never rewrite `createdAt`. The current `routineRepository.set` unconditionally stamps `updatedAt` on every write and callers pass `createdAt` only on create. We tighten this:

- **Create** (`routineService.create`): stamp both `createdAt` and `updatedAt` = `serverTimestamp()`.
- **Update** (`routineService.update`, completion toggles): read existing, compute whether any tracked field changed; if nothing changed, **return existing without writing** (no `updatedAt` churn — this is what makes an idempotent duplicate completion a true no-op). If changed, `set(..., {merge:true})` with `updatedAt` only; never include `createdAt`.
- Completion doc: on first write set `createdAt`; on status change set `updatedAt` + `completedAt`; on a repeat of the same status, no write.

### 2.5 Frontend `Routine` model (sync)

Extend `frontend/lib/models/routine.dart`:
- Add fields `endTime? , repeat (RoutineRepeat enum once|daily|selected), date?, priority (RoutinePriority enum low|medium|high), isRequiredForStreak (default true), linkedWorkoutId?, isActive (default true)`.
- Add `RoutineRepeat` and `RoutinePriority` enums with `wire`/`label`/`fromWire`, mirroring the `RoutineCategory` style.
- Extend `fromJson` with defaults for every new field (so old cached JSON still parses): `repeat` defaults by presence of `date`/`repeatDays`, `priority→medium`, `isRequiredForStreak→true`, `isActive→true`.
- Extend `toUpsertJson` to emit the new fields; emit `date` only when `repeat == once`.
- Keep `RoutineDay` as-is; it gains no new fields (it reflects a single day's scheduled routines + completion).

---

## 3. Firestore collection paths & indexes

- Routines: `users/{uid}/routines/{routineId}` (existing).
- Completions: `users/{uid}/routineCompletions/{dateKey__routineId}` (existing).
- Workouts: `users/{uid}/workouts/{workoutId}` (existing).
- Game history (untouched): `users/{uid}/gameHistory/{...}`.

Security: `firebase/firestore.rules` already grants owner-only `read, write` on `users/{uid}/{document=**}`. No new collections are introduced, so **no rule change is needed**.

Indexes (`firebase/firestore.indexes.json`):
- **`routineCompletions` monthly range — NO new index needed** (resolves review Finding 6). The monthly range query is `where('dateKey','>=',start).where('dateKey','<=',end)` — a single-field range on one field. Firestore auto-creates single-field ascending indexes, and the existing composite `dateKey ASC, routineId ASC` additionally serves any `dateKey`-prefixed range. Do **not** add an explicit `routineCompletions: dateKey ASC` entry; it is unnecessary.
- **Workout streak — ADD one composite index** (resolves review Finding 2). The streak query filters completed workouts and ranges on the completion day: `where('status','==','completed').where('completedDateKey','>=',start).where('completedDateKey','<=',end)`. An equality predicate (`status`) combined with a range predicate (`completedDateKey`) on two different fields **requires a composite index** — a single-field `completedDateKey` index would throw `FAILED_PRECONDITION`. Add exactly:
  ```json
  { "collectionGroup": "workouts", "queryScope": "COLLECTION",
    "fields": [ { "fieldPath": "status", "order": "ASCENDING" },
                { "fieldPath": "completedDateKey", "order": "ASCENDING" } ] }
  ```
  This mirrors the already-present `workouts: status ASC, startedAt DESC` composite. **Decision — keep the `status==='completed'` predicate** (rather than dropping it and relying on "only completed workouts have `completedDateKey`"): it makes the query intent explicit, defends against any future code path that might stamp `completedDateKey` in another state, and matches the existing status-prefixed index convention. The exact query and exact index above are locked. Do **not** touch project values / `google-services.json`.

---

## 4. Backend — routes, services, validators

All new routes mount under the existing `routine.routes.ts` (or a sibling `streak.routes.ts`), each starts with `router.use(authenticate)`, every controller derives UID via `requireUid(req)` (never a client UID), uses `validate({...})` Zod middleware, `asyncHandler`, and the `ok(res, data, status?)` envelope. Ownership is structural (`users/{uid}/...`).

### 4.1 Validator changes (`backend/src/validators/routineValidators.ts`)

Extend `routineUpsertSchema` additively:
```
repeat: z.enum(['once','daily','selected']).default('daily'),
date: commonSchemas.dateKey.optional(),      // required-if repeat==once (refine)
endTime: commonSchemas.timeOfDay.optional(),
priority: z.enum(['low','medium','high']).default('medium'),
isRequiredForStreak: z.boolean().default(true),
linkedWorkoutId: z.string().min(1).max(200).optional(),
category: z.enum(ROUTINE_CATEGORIES)          // enum auto-extended in domain.ts
```
Add `.superRefine` enforcing: `repeat==='once'` ⇒ `date` present; `repeat==='selected'` ⇒ `repeatDays` non-empty; `endTime` (if present) ≥ `time`.

> Note on the `endTime >= time` check (resolves review Finding 5). `commonSchemas.timeOfDay` is a zero-padded 24-hour `HH:mm` string (`"05:30"`, `"18:00"`), so a lexicographic `endTime >= time` comparison is **correct by construction** — zero-padded fixed-width 24h strings sort identically to their chronological order. To make the intent unmistakable and prevent an implementer from "fixing" it into a buggy parse, implement the comparison through a tiny pure helper `toMinutes(hhmm: string): number` (`h*60+m`) and compare numerically (`toMinutes(endTime) >= toMinutes(time)`); add a one-line comment that the lexical compare would also be safe. `toMinutes` is trivially unit-tested.

Keep `enabled`/`notificationEnabled` as-is. `isActive` is **not** accepted from the upsert body (server-managed; pause uses `enabled`, delete uses `isActive`).

Add query/param schemas:
- `monthQuerySchema = z.object({ year: z.coerce.number().int().min(2000).max(2100), month: z.coerce.number().int().min(1).max(12) })` — the public query schema for `GET /api/routines/month` and the optional `?year&month` on `/api/streaks/summary`.
- `rangeBounds` (internal, not a public route schema): a shared helper validates/derives `{ start, end }` from year/month and enforces span ≤ 62 days as a defensive cap before the Firestore range read. **The span cap is enforced by throwing a `BadRequestError` (→ HTTP 400) from this helper, not by a Zod refine.** Rationale: `year` and `month` are each individually valid (that is what `monthQuerySchema`'s Zod rules check, yielding 422 on a bad `year`/`month`); the span is a *cross-field, derived* runtime invariant computed after the schema passes, so it belongs in the aggregator path as an `AppError` whose `statusCode` the error handler honors (400), not in `.parse()` (which would emit a `ZodError` → 422). This keeps a clean split: schema-shape failures = 422, derived runtime guard = 400. (This is the former `routineRangeQuerySchema`, now an internal input to the aggregator per §4.2, not an endpoint.)

### 4.2 Routes (final list)

Existing (behavior adjusted where noted):
- `GET /api/routines` — list routines. **Change:** exclude `isActive === false` by default; accept `?includeArchived=true` for admin/debug (not used by UI).
- `GET /api/routines/today?date=YYYY-MM-DD` — single day with completion (reused for day-detail of any historical/future date).
- `POST /api/routines` — create.
- `PUT /api/routines/:id` — edit (also used for pause/resume by toggling `enabled`).
- `DELETE /api/routines/:id` — **soft delete** (sets `isActive:false`; keeps completions). See §4.3.
- `POST /api/routines/:id/complete` — write/update a `RoutineCompletion` for a `dateKey` (idempotent).

New:
- `POST /api/routines/:id/uncomplete` *(or reuse `/complete` with `status:'pending'`)* — set completion back to `pending`, clear `completedAt`. **Decision:** reuse the existing `/complete` endpoint with `status:'pending'` (the frontend already sends `status`); no new route needed. This keeps one write path and one idempotency key.
- `GET /api/routines/month?year=&month=` — **the single endpoint the calendar UI calls.** Computes `start`/`end` for a calendar month server-side (handles leap year / month length), returns the per-day array `[{ dateKey, total, completed, completionPercent, state, isRoutineStreakDay }]` (`state ∈ full|partial|missed|none`) plus `monthRoutineProgress` (days fully completed / days with required routines).
- `GET /api/streaks/summary` — the single streak endpoint consumed by the client façade:
  ```
  {
    routine: { current, best },
    workout: { current, best },
    monthlyRoutineProgress?: {...},   // optional, when ?year&month passed
    monthlyWorkoutProgress?: {...}
  }
  ```
  Accepts optional `?year=&month=` to also return the two monthly progress objects in one round-trip. The **streaks UI** (streak details page, dashboard banner) calls this; the **calendar UI** calls `/routines/month`.

> **Endpoint consolidation (resolves review Finding 7).** There is no `/api/routines/range` endpoint. Earlier drafts proposed both `/range` and `/month`; to avoid three code paths that can drift, we expose exactly one per-day month aggregate (`/routines/month`) and one streak endpoint (`/streaks/summary`). Both `/routines/month` and the `monthlyRoutineProgress` portion of `/streaks/summary` MUST call the **same single internal aggregator function** (`getMonthlyRoutineProgress`, §5.4) so per-day state is computed in exactly one place. The month-bounds helper (year/month → `start`/`end`, leap-year aware) is likewise one shared function. `routineRangeQuerySchema` in §4.1 becomes the internal input to that aggregator (start/end are derived from year/month), not a public route.

### 4.3 Soft-delete (fixes the history-destroying bug)

`routineService.delete` is rewritten: it **must not** call `deleteCompletionsForRoutine`. Instead it sets `isActive:false` (+ `updatedAt`). `deleteCompletionsForRoutine` remains in the repo but is no longer called by delete (kept only for the account-wipe path, which already uses `recursiveDelete`). Rationale (state explicitly, per audit §5): streak/calendar reconstruct each historical day's scheduled-required set from the **current** routine definitions; archiving (not purging) keeps the routine's name/`repeatDays`/`isRequiredForStreak` available so historical days render correctly and completions are never orphaned.

#### Read-path filter rule (resolves review Finding 1 — HIGH)

Soft-delete sets `isActive:false` but leaves `enabled` untouched, so the **today / day-detail / calendar read paths must explicitly exclude `isActive===false`** — otherwise a "deleted" routine keeps appearing in today's list and inflates the daily-progress count. The current `getDay` filters only `enabled !== false`; that is insufficient and must be changed. Define one filter rule, applied in `getDay` and in the month aggregator (`getMonthlyRoutineProgress`), keyed off whether the day is in the present/future or the past relative to the server's local `today`:

- **Day `>= today` (present/future scheduling):** include a routine iff
  `r.isActive !== false && r.enabled !== false && isRoutineOnDate(r, dateKey)`.
  Archived (deleted) and paused routines are both hidden — the user should not see or be scored on a routine they deleted or paused.
- **Day `< today` (historical reconstruction):** include a routine iff
  `r.isActive !== false ? (r.enabled !== false && isRoutineOnDate(r, dateKey)) : isRoutineOnDate(r, dateKey)` is **not** the rule — see the correction in §5.1. For historical days the authoritative signal is the **stored completion records**, not a replay of current `enabled`/`isActive`. The day's rendered goal list (day-detail) shows every routine that `isRoutineOnDate(r, dateKey)` and for which a completion record exists OR the routine is still `isActive!==false`, so an archived routine's past completions still render with its last-known name. Historical qualification for the streak is governed solely by stored completions (§5.1), which freezes history and prevents a present-day delete/pause from retroactively rewriting past days.

In short: **present/future reads exclude `isActive===false` and `enabled===false`; historical reads are driven by stored completion records (and show archived routines' past completions for display).** `getDay` must therefore be given the current `todayKey` so it can branch, and is listed as an edited method in §13.

### 4.4 `routineCalc.ts` extension

`isRoutineOnDate` gains the `repeat`/`date` dimension. New signature keeps back-compat:
```
isRoutineOnDate(routine: { repeat?: string; date?: string; repeatDays?: number[] }, dateKey: string): boolean
```
Logic: `repeat==='once'` ⇒ `routine.date === dateKey`; `repeat==='daily'` or absent-with-empty-repeatDays ⇒ true; `repeat==='selected'` (or legacy with non-empty repeatDays) ⇒ `repeatDays.includes(weekdayForDateKey(dateKey))`. Callers (`getDay`, range aggregation, streak) pass the routine object. Keep `weekdayForDateKey` and `completionPercent` unchanged. **Unit-tested** in `routineCalc.test.ts` (once/daily/selected, leap-day boundary, empty repeatDays).

---

## 5. Streak computation (backend, single source of truth)

New module `backend/src/utils/routineStreakCalc.ts` plus reuse of `streakCalc.computeStreak`. All inputs are **persisted completion records only** — never auth/login/app-open/game data.

### 5.1 Routine streak

**Lookback window (locked — resolves review Finding 3).** Both current and best Routine-streak calcs operate over a single fixed rolling window: the last **`STREAK_LOOKBACK_DAYS = 366`** days ending at `today` (named constant in `routineStreakCalc.ts`, shared by current and best, and reused by the workout calc §5.2). 366 covers a full year including a leap day. Consequence stated honestly: "best streak" means **best within the last 366 days** — a longer run entirely older than 366 days is not surfaced. This bounds Firestore read volume (precedent: `gameService.summary` caps its history read), keeps a single read window feeding both numbers, and makes the calc deterministic/testable. There is no "or all records" alternative.

**Historical qualification is frozen by stored completion records (resolves review Finding 4 — MEDIUM).** `enabled` (pause) and `isActive` (archive) are single current booleans with no per-day history, so replaying them against past days would let a *present-day* pause/delete retroactively alter past streak qualification (e.g. silently resurrect a broken streak). We therefore make the rule computable purely from data that was true at the time: **a historical day's qualification is driven by the `RoutineCompletion` records that exist for that day, not by replaying the current `enabled`/`isActive` flags.** Pause/archive are **forward-only**: they change scheduling of new days from "now" onward and never rewrite history.

`qualifyingRoutineDays(routines, completionsByDateKey, today): string[]` over the 366-day window:

1. **Determine the required-and-scheduled set per day**, branching on past vs present/future (this is the computable restatement; it uses stored records for the past):
   - **Day `>= today`:** `required = routines.filter(r => r.isActive !== false && r.enabled !== false && r.isRequiredForStreak === true && isRoutineOnDate(r, day))`. (Paused/archived routines are not scheduled going forward, so they cannot block the streak — matches "optional/none should not break the streak" and the forward-only pause semantics.)
   - **Day `< today` (historical):** the required set is reconstructed as the routines that were actually tracked that day — i.e. `required = routines.filter(r => r.isRequiredForStreak === true && isRoutineOnDate(r, day) && aCompletionRecordExistsFor(r.id, day))` **unioned with** any `isActive!==false && isRequiredForStreak` routine scheduled that day. In practice the authoritative check reduces to: *for every routine that was required and scheduled that day and for which the system wrote (or should have written) a completion record, a `completed` record must exist.* Because a present-day pause/archive does not delete past completion records, those past days keep their original qualification — history is frozen. A routine paused today contributes to a past day only through the completion records already written for that day; it is never retro-removed and never retro-added.
2. If `required.length === 0` for a day ⇒ day is **neutral** (⚪): **not** a qualifying streak day and **not** a break — it is skipped (the cursor keeps looking). (Optional/none days do not break the streak.)
3. Else the day **qualifies** iff every member of `required` has a `completed` completion for that `dateKey`.

`calculateRoutineStreak(today)`: feed qualifying dateKeys into `computeStreak(dateKeys, today)` — strict consecutive-day count (no grace day). Because neutral days are skipped rather than counted, a day with zero required tasks neither extends nor resets the streak, and `computeStreak`'s "ends today or yesterday" tolerance is preserved. The neutral-day/cursor interaction is a **required unit-test case** (§11), not an assumption.

> Edge rule stated explicitly: a day in the middle of the window with required tasks that are not all completed is a **break** (missed), so the current streak ends there (spec item 12 — Thursday restarts at 1). A day with zero required tasks is transparent.

`calculateBestRoutineStreak(today)`: scan the **same 366-day** qualifying-day sequence and return the longest consecutive run (skipping neutral days). Same window, same records, same aggregator input as `calculateRoutineStreak` — only the reduction differs (longest-run vs ends-today).

### 5.2 Workout streak

Audit §4: the authoritative "completed" signal is a workout doc reaching `status === 'completed'`. There is no `dateKey` on workouts, so **in `workoutService.transition`, when `to === 'completed'`, additionally stamp `completedDateKey`** (local `yyyy-MM-dd` derived from the completion moment) on the workout doc. This is additive and only on completion; it does not alter workout internals.

`calculateWorkoutStreak` / `calculateBestWorkoutStreak`: over the same fixed `STREAK_LOOKBACK_DAYS = 366` window (shared constant, §5.1), query completed workouts via `listCompletedInRange` — `where('status','==','completed').where('completedDateKey','>=',start).where('completedDateKey','<=',end)` using the composite index added in §3 — map `completedDateKey`, de-dupe, and feed to `computeStreak(dateKeys, today)` (current) / longest-run scan (best). "Best workout streak" is likewise "best within the last 366 days." Independent from routine streak by construction (disjoint record sets; the only coupling is the one-directional link in §5.3).

### 5.3 Routine ↔ workout linkage (one-directional, explicit)

Spec item 13 + audit §5: a routine with `linkedWorkoutId` set and `isRequiredForStreak:true` is auto-satisfied **only** when that specific linked workout completes. Implementation (server-side, in `workoutService.transition` on `to==='completed'`):
1. Query `routines` where `linkedWorkoutId === id` and `isActive !== false`.
2. For each, write a `RoutineCompletion` (`dateKey = completedDateKey`, `status:'completed'`, `completedAt`) via the same idempotent path.
This writes a routine completion from a workout completion **only for the explicitly linked routine** — never auto-linking unrelated workouts. Completing routine tasks never writes workout completions (no reverse path exists). The two streaks remain independent; the only coupling is this explicit per-routine link.

### 5.4 Monthly progress

- `getMonthlyRoutineProgress(year, month)`: the **single internal aggregator** (resolves review Finding 7) called by both `GET /api/routines/month` and the `monthlyRoutineProgress` field of `GET /api/streaks/summary`. Returns the per-day aggregate over the month (`dateKey`, `total`, `completed`, `completionPercent`, `state`, `isRoutineStreakDay`), plus totals (`daysWithRequired`, `daysFullyCompleted`, `routineStreakDaysInMonth`). It derives month bounds via the one shared leap-year-aware `rangeBounds` helper and applies the exact present/future-vs-historical filter rule from §4.3/§5.1 so per-day `state` is computed in exactly one place.
- `getMonthlyWorkoutProgress(year, month)`: count of days with a completed workout, list of `completedDateKey`s in the month.
Both are pure functions over fetched records ⇒ **Vitest unit tests** (`streakCalc.test.ts` neighbor + new `routineStreakCalc.test.ts`), covering: no routines, all optional, mixed required/optional, missed middle day resets, month boundary, leap year, linked-workout auto-complete.

---

## 6. Workout backend touch points

- `workoutService.transition`: on `to==='completed'`, add `completedDateKey` to the patch (local day) and run the linked-routine auto-complete (§5.3). PR detection stays. On `cancelled`, do **not** set `completedDateKey`.
- `workoutRepository.list`: add support for a `completedDateKey` range query (new optional opts `{ completedFrom, completedTo }`) used by the streak calc, or add a dedicated repo method `listCompletedInRange(uid, start, end)`. Decision: **add `listCompletedInRange`** to keep `list`'s signature stable.
- No change to `canTransition`, set toggling, or completion immutability. "Start"/"add set"/"page open" still never qualify (audit §4) — only `transition→completed` writes `completedDateKey`.

---

## 7. Frontend — providers, screens, routing

### 7.1 Models & repository

- `routine_repository.dart`: extend `toUpsertJson`/`fromJson` via the model; add `getMonth(year,month)` → `MonthData` (per-day `List<DayProgress>` + month totals), and keep `getDay`/`setCompletion`. (No `getRange`; the calendar uses `/routines/month` only — review Finding 7.) Add a `StreakRepository` (new `features/streaks/data/streak_repository.dart`) hitting `/api/streaks/summary`. All return `Result<T>` (`Success`/`Err(Failure)`), matching existing convention.
- New DTOs: `DayProgress { dateKey, total, completed, completionPercent, state (enum full|partial|missed|none), isStreakDay }`, `StreakSummary { routineCurrent, routineBest, workoutCurrent, workoutBest, monthlyRoutine?, monthlyWorkout? }`.

### 7.2 Providers (Riverpod)

- Keep `routinesProvider`, `routineTodayProvider`, `routineRepositoryProvider`.
- Add `routineMonthProvider.family<MonthData, (int year,int month)>` (calendar), `dayDetailProvider.family<RoutineDay, String dateKey>` (reuses `getDay`), `streakSummaryProvider` (FutureProvider over `StreakRepository`), and `dashboardStreakPrefProvider` (§7.6).
- Add an **optimistic completion** controller for the today list matching the `WaterQuickWidget` pattern (audit §10): local optimistic status flip on tap → progress bar/percent update instantly → `setCompletion` round-trip → on success invalidate `routineTodayProvider`/`routineMonthProvider`/`streakSummaryProvider`; on failure roll back + retry SnackBar.

### 7.3 Daily Routine / Goals page (upgrade `routine_screen.dart` in place)

Audit §7 decision adopted: **upgrade the existing `RoutineScreen`**, do not add a 6th bottom-nav tab (shell hard-codes 5). Layout top→bottom:
1. **Today's progress card** (existing) — `3/5 completed`, remaining, `completionPercent`, progress bar. Reuse existing card; add "remaining" count.
2. **Task list** with completion control — `CheckboxListTile` (existing), now showing priority chip, category icon, time range (`time`–`endTime`), and a required/optional indicator. Completion uses the optimistic controller. Opening the page must **not** auto-complete anything (completion only on explicit tap).
3. **Add-routine FAB** → `RoutineEditorScreen` with the **full field set**: title, description, category (incl. new values), date (when `once`), start time, end time, repeat (once/daily/selected with weekday chips), reminder toggle, priority, `Required for streak` toggle (default ON), optional `linkedWorkoutId` picker (choose from the user's workout templates/sessions).
4. **Management list** (existing) — edit / delete (now archive) / pause-resume (toggle `enabled`) via the popup menu; add explicit Pause/Resume menu items.
5. **Empty state** — reuse `EmptyView` from `widgets/state_views.dart`.
6. **Monthly calendar** section (below the daily section) and a link into **day detail** (§7.4).

### 7.4 Monthly calendar + day detail

New widget `features/routine/presentation/widgets/routine_calendar.dart`:
- Compact selectable **tiles** in a 7-column grid (Mon–Sun header), seat-selection concept (not literal ticket assets). Reuse `AppColors`/`AppSpacing`; map states to theme colors: 🟢 full → success/primary, 🟡 partial → amber, 🔴 missed → error, ⚪ none → surfaceVariant, 🔥 streak day → `AppColors.fireGradient` accent on top of the tile.
- Month nav (prev/next) driven by `routineMonthProvider.family`. Leading blank cells for the first weekday; correct day count per month (leap year via backend `/month`).
- Tapping a tile pushes a **day-detail** view (`routine_day_detail_screen.dart`) that renders `getDay(dateKey)`: date header, goal count, per-goal completed/incomplete rows (read-only indicators), `X/Y completed` + percentage. Historical dates are viewable; the detail view performs **no writes** on open.

### 7.5 Streak details page

New `features/streaks/presentation/streak_details_screen.dart`: shows both streaks (current + best for Routine and Workout) from `streakSummaryProvider`, plus the current month's routine/workout progress. Reachable from the dashboard streak card and from the Routine page. Pure display.

### 7.6 Dashboard streak preference (persistence)

Audit §8 decision adopted: **SharedPreferences via a `StateNotifierProvider`**, mirroring `theme_provider.dart` exactly (UI-only preference, no cross-device requirement).
- New `features/settings/dashboard_streak_pref.dart`: `enum DashboardStreakPref { routine, workout, both, none }`; `DashboardStreakPrefNotifier extends StateNotifier<DashboardStreakPref>` reading/writing key `dashboard_streak_pref` via `sharedPreferencesProvider`; default `both`.
- Settings UI: add a `ListTile` with a dropdown/radio in `profile_screen.dart`'s existing Settings `Card`, same structure as the Theme dropdown.

### 7.7 Dashboard banner rework

`dashboard_screen.dart`: replace `_StreakBanner`'s data source (currently `gameSummaryProvider.streak`) with `streakSummaryProvider` + `dashboardStreakPrefProvider`:
- `routine` ⇒ `🔥 N Day Routine Streak`.
- `workout` ⇒ `🔥 N Day Workout Streak`.
- `both` ⇒ two fire cards (`🔥 N Routine`, `🔥 M Workout`).
- `none` ⇒ hide the banner.
Reuse the existing fire-gradient `_StreakBanner` widget (duplicate/parametrize for two values). Add the new providers to the existing `RefreshIndicator.onRefresh` invalidation set. **Do not touch** `WaterQuickWidget`, `RemindersQuickWidget`, nutrition, AI insight, or `TodayRoutineCard`'s water/reminder behavior. The morning-game streak (`gameSummaryProvider`) stays wired to its own feature; it is simply no longer the dashboard's generic banner.

### 7.8 Routing

Routine page stays reachable from Profile → "Daily routine" (and a new dashboard shortcut). GoRouter stays single-shell/5-tab; new screens (day detail, streak details, editor) are pushed as sub-routes/`MaterialPageRoute` consistent with the current `RoutineEditorScreen` push. No change to splash/auth redirects or `home_shell` startup side-effects (preserve FCM/reschedule — audit §2/§13).

---

## 8. Centralized StreakService contract (client façade)

New `frontend/lib/features/streaks/streak_service.dart` — the **single** place any screen reads streak/progress data from (Home/Routine/Workout/Profile all consume this, no duplicated logic). It delegates to the backend; it does not reimplement the rule.

```dart
abstract class StreakService {
  Future<Result<int>> calculateRoutineStreak();        // current
  Future<Result<int>> calculateWorkoutStreak();         // current
  Future<Result<int>> calculateBestRoutineStreak();
  Future<Result<int>> calculateBestWorkoutStreak();
  Future<Result<MonthlyProgress>> getMonthlyRoutineProgress(int year, int month);
  Future<Result<MonthlyProgress>> getMonthlyWorkoutProgress(int year, int month);
}
```
Implementation fetches `/api/streaks/summary` (optionally with `year`/`month`) once and derives the six getters from the cached `StreakSummary` within a request, so a dashboard render that needs both streaks is a single round-trip. Exposed via `streakSummaryProvider`/`streakServiceProvider`. **Contract invariant:** every value originates from persisted `routineCompletions`/completed `workouts` on the backend; the client never computes a streak from auth, app-open, or local state.

---

## 9. Migration strategy

- **Routine definitions:** no migration. New fields are optional with safe defaults; existing docs read correctly (`repeat` inferred, `priority→medium`, `isRequiredForStreak→true`, `isActive→true`).
- **Completions:** no migration. Existing docs lack `completedAt`/`createdAt`; the day-detail/calendar/streak logic treats `status==='completed'` as the signal and tolerates missing `completedAt` (ordering/history still works via `dateKey`). New writes populate the fields.
- **Login/app-open streak:** the audit confirmed **no `lastLoginDate`/`appOpen`/`firebaseAuthDate` streak exists** to carry forward — nothing to delete. The dashboard simply stops showing the morning-game streak as the generic streak and shows the new activity-based streaks instead. We **do not** seed Routine/Workout streak values from any login/game data; both start from genuine completion records only. Genuine historical activity (existing `routineCompletions`, completed `workouts`, `gameHistory`) is preserved untouched.
- **Versioning:** no explicit schema-version field is required because every change is additive and default-safe. If a future rule change needs it, add a `schemaVersion` on the routine doc; not needed now (state this so the implementer doesn't invent one).

---

## 10. Error handling (per operation)

| Operation | Failure condition | Recoverable? | Caller receives | Logged? |
| --- | --- | --- | --- | --- |
| Create/update routine | Zod validation fails (type/shape/enum/limit) | Recoverable | **422** via `validate` middleware → `ZodError` → `errorHandler` emits `{code:'VALIDATION_ERROR', message:'Request validation failed', details: err.flatten()}` | request-level (existing error middleware) |
| Create/update routine | `repeat==once` without `date`, `endTime<time`, `selected` without days (`.superRefine`) | Recoverable | **422** — these refinements raise a `ZodError` inside `.parse()`, same path as above (not a separate 400 path) | middleware |
| Update/complete | routine not found / archived | Recoverable | 404 `NotFoundError` | middleware |
| Complete (idempotent) | same status re-sent | n/a (no-op) | 200 with existing record, no write | not logged |
| Delete (soft) | not found | Recoverable | 404 | middleware |
| Delete (soft) | success | — | 200 `{deleted:true}`; completions untouched | — |
| Month aggregate | bad `year`/`month` shape (out of range, non-numeric) | Recoverable | **422** — `monthQuerySchema` Zod failure via `validate` middleware | middleware |
| Month aggregate | derived span > 62 days (cross-field runtime guard) | Recoverable | **400** — `rangeBounds` helper throws `BadRequestError`; `errorHandler` honors `AppError.statusCode` (400). Deliberately **not** a Zod refine (would be 422) — see §4.1 | middleware |
| Streak summary | Firestore read error | Fatal to request | 500 envelope | server error log (no secrets) |
| Workout transition→completed | linked-routine auto-complete write fails | **Non-fatal** (best-effort, like PR detection) | workout completion still returns 200 | server warn log; streak self-heals on next read |
| Frontend any repo call | `Failure` thrown by `ApiClient` | Recoverable | `Err(Failure)`; UI shows `ErrorView`/retry SnackBar | — |
| Frontend optimistic completion | network error | Recoverable | roll back optimistic state + retry SnackBar | — |

Validation rules for external inputs are the Zod schemas in §4.1 (required/optional, type, limits). **Status-code contract (verified against `backend/src/middleware/validate.ts` + `errorHandler.ts`):** any Zod schema/`superRefine` failure propagates as a `ZodError` and the central `errorHandler` emits **HTTP 422** (`code:'VALIDATION_ERROR'`); `NotFoundError` → 404 and any other `AppError` → its own `statusCode` (e.g. the span-cap `BadRequestError` → 400); unhandled errors → 500. Implementers must encode **422** (not 400) in integration tests for malformed create/update bodies and bad month params, and **400** only for the derived span-cap guard. The idempotency invariant (no duplicate completion) is owned by the **repository layer** via the deterministic `${dateKey}__${routineId}` doc id + merge write; the "which day qualifies / strict reset" invariant is owned by the **streak util layer** (pure, tested) so it has exactly one implementation.

---

## 11. Testability

- **Unit (Vitest, backend):** `routineCalc.isRoutineOnDate` (once/daily/selected, legacy empty, leap day); `toMinutes` + `endTime>=time` refine (Finding 5); `routineStreakCalc.qualifyingRoutineDays` + `calculateRoutineStreak`/`Best` (no routines, all optional, mixed required/optional, missed-middle reset, **neutral-day skip vs `computeStreak` today/yesterday cursor** — required case per review, month/leap boundary, **366-day window edge** — a run older than 366 days is excluded, Finding 3); **historical-freeze test** (Finding 4): pausing/archiving a routine today does not change the qualification of a past day driven by its stored completion records; **read-path filter test** (Finding 1): a soft-deleted (`isActive:false`) routine is excluded from `getDay`/month aggregation for days `>= today`; workout streak from `completedDateKey` via the composite-index query (Finding 2); linked-workout auto-complete writes exactly the linked routine. These pure functions are the real verification, not grep.
- **Integration-ish (backend):** service tests over the repos (where the existing suite does so) for soft-delete-keeps-completions and idempotent completion no-op. **Status-code assertions** must match the real pipeline (§10): malformed create/update body or bad `year`/`month` ⇒ assert **422** (`ZodError` via `errorHandler`); a month request whose derived span exceeds 62 days ⇒ assert **400** (`BadRequestError` from `rangeBounds`); not-found ⇒ 404. These guard against regressing the error contract an implementer encodes into client branching.
- **Flutter (`flutter test`):** model `fromJson`/`toUpsertJson` round-trip with new fields + legacy JSON; calendar state→color mapping; optimistic completion roll-back. Widget-level day-state rendering.
- A design red flag check: the streak rule lives in one pure module consumed everywhere ⇒ easy to test and no duplicated logic to drift. If any screen needed to recompute a streak locally, that would signal a design problem — the façade prevents it.

Verification commands (from audit §0): `cd backend && npm test && npm run build && npm run lint`; `cd frontend && flutter analyze && flutter test`. No APK build, no `flutter pub get` for new packages.

---

## 12. Edge cases

- **No routines / 1 / 20+:** today card and calendar render; empty→`EmptyView`; 20+ scrolls; streak with no required tasks any day = 0 (all neutral). Range query capped at 62 days bounds reads.
- **Recurring (daily/selected):** scheduled set reconstructed per day from current definition via `isRoutineOnDate`.
- **Selected-weekdays:** `repeatDays.includes(weekday)`; weekday derived in local time (`weekdayForDateKey`).
- **Deleted-routine-with-history:** soft-delete (`isActive:false`) keeps completions; excluded from today/management and from present/future scheduling (Finding 1 filter); past days remain governed by their stored completion records (frozen history, Finding 4), and day-detail still shows the routine's past completions under its last-known name.
- **Edited recurring routine:** present/future days use the current definition; past days are frozen by stored completion records keyed by `routineId` (Finding 4); changing `isRequiredForStreak` or `enabled`/`isActive` affects only days `>= today`, never rewrites past qualification.
- **Pause mid-window:** pausing today does not retroactively neutralize past missed-required days (would otherwise resurrect a broken streak); pause is forward-only (Finding 4).
- **Streak older than 366 days:** not surfaced — current and best are both bounded to the rolling 366-day window (Finding 3).
- **Timezone / midnight transition:** all dateKeys are local `yyyy-MM-dd` (`weekdayForDateKey`, controller `todayKey`, `completedDateKey` all local). A completion near midnight is attributed to the local day it was tapped. Documented single-timezone assumption (app uses device local time, consistent with existing water/food `dateKey`).
- **Duplicate completion (idempotent):** deterministic doc id + status-unchanged no-op ⇒ no duplicate, no `updatedAt` churn.
- **Offline completion then sync:** same deterministic id ⇒ sync merges into the one doc; no duplicate. Optimistic UI reconciles on reconnect.
- **Logout/login:** no streak side-effect — login never writes a completion (hard rule). `home_shell` startup (FCM/reschedule) untouched and writes no streak.
- **New month:** calendar nav fetches that month; `/month` computes correct bounds.
- **Leap year:** `/month` + `isRoutineOnDate` handle Feb 29; covered by unit test.
- **Missed day:** required-but-incomplete day breaks the streak (restarts next qualifying day at 1); strict, no grace day.
- **Future/past date:** day-detail/`getDay` works for any date; future dates show scheduled routines with `pending` status and no writes; completion only allowed/meaningful for dates the user actually acts on.
- **Partial day:** `state=partial` (🟡) when `0 < completed < scheduled`; counts as missed for the streak if any **required** task is incomplete.

---

## 13. Integration points (every existing file touched/extended)

Backend:
- `backend/src/models/domain.ts` — extend `ROUTINE_CATEGORIES` (add `morning,health,personal,other`).
- `backend/src/validators/routineValidators.ts` — extend `routineUpsertSchema`; add range/month schemas.
- `backend/src/utils/routineCalc.ts` — extend `isRoutineOnDate` for `repeat`/`date`.
- `backend/src/utils/streakCalc.ts` — reused as-is (no edit); new `routineStreakCalc.ts` + `workoutStreakCalc` helper added alongside.
- `backend/src/services/routineService.ts` — create/update stamp write-once `createdAt`/idempotent `updatedAt`; **delete→soft-delete (stop purging completions)**; **`getDay` edited** to take `todayKey` and apply the present/future-vs-historical read-path filter (exclude `isActive===false` and `enabled===false` for days `>= today`; drive historical days from stored completions — review Finding 1); add single `getMonthlyRoutineProgress` aggregator (used by `/routines/month` and `/streaks/summary`); completion sets `completedAt`.
- `backend/src/repositories/routineRepository.ts` — add `dateKey` range query (no new index needed, Finding 6); completion write adds `completedAt`/`createdAt`; `deleteCompletionsForRoutine` retained but unused by delete.
- `backend/src/services/workoutService.ts` — `transition→completed` stamps `completedDateKey` + linked-routine auto-complete (best-effort).
- `backend/src/repositories/workoutRepository.ts` — add `listCompletedInRange`.
- `backend/src/controllers/routineController.ts` — add the `/routines/month` handler; delete returns soft-delete result.
- new `backend/src/controllers/streakController.ts` + `backend/src/routes/streak.routes.ts`; mount in `backend/src/app.ts`. Single `/routines/month` route added to `routine.routes.ts` (no `/routines/range` — Finding 7).
- `firebase/firestore.indexes.json` — **add exactly one** composite index `workouts: status ASC, completedDateKey ASC` for the workout-streak query (Finding 2). No `routineCompletions` index change (the existing `dateKey ASC, routineId ASC` + auto single-field indexes cover the monthly range — Finding 6). `firebase/firestore.rules` — **no change** (owner-only wildcard already covers all per-user data).

Frontend:
- `frontend/lib/models/routine.dart` — add fields + `RoutineRepeat`/`RoutinePriority` enums + category sync.
- `frontend/lib/features/routine/data/routine_repository.dart` — add `getMonth`; new DTOs (`DayProgress`, `MonthData`).
- `frontend/lib/features/routine/providers/routine_providers.dart` — add month/day-detail/streak/pref providers + optimistic completion controller.
- `frontend/lib/features/routine/presentation/routine_screen.dart` — upgrade (progress card, optimistic completion, management with pause/resume, calendar section).
- `frontend/lib/features/routine/presentation/routine_editor_screen.dart` — full field set (date, end time, repeat mode, priority, required-for-streak, linked workout).
- new `routine_calendar.dart`, `routine_day_detail_screen.dart`, `features/streaks/*` (service, repository, providers, details screen).
- new `features/settings/dashboard_streak_pref.dart`; `profile_screen.dart` — add preference tile; profile already renders a day-streak stat → point it at the new preference-driven value.
- `frontend/lib/features/home/presentation/dashboard_screen.dart` — rework `_StreakBanner` data source + add providers to `onRefresh`. **No change** to water/reminder/nutrition/AI blocks.
- `frontend/lib/features/notifications/reminder_scheduler.dart` — honor new `repeat`/`date`/`endTime`: schedule one-shot for `once`, weekly for `selected`, daily for `daily`; keep cancel-all-then-reschedule idempotency. Reuse `NotificationService`; **no new scheduler/package**.
- `frontend/lib/core/theme/theme_provider.dart` — pattern reference only (not edited).

### Hard constraints confirmed respected
- No APK build; no new packages / dependency upgrades; Riverpod/GoRouter/Dio/Firestore/`flutter_local_notifications`/`shared_preferences` reused.
- `google-services.json`, `firebase_options.dart` project values, `applicationId`, signing untouched.
- No mock/fake data; no hardcoded UIDs; every UID from the verified Firebase token via `requireUid` (ownership structural under `users/{uid}`).
- Streaks computed only from persisted completion records; login/app-open/splash/auth/game never feed the two new streaks.
- Preserved prior fixes left intact: `home_shell` auth/FCM/reschedule, AI history, alarm/game state machine, morning-challenge streak, water optimistic widget, reminder scheduler idempotency.
- Deleting a routine never destroys historical `routineCompletions` (soft-delete).

---

## 14. Decision log (chosen, not left open)

1. **Extend routine, not a new "goals" sibling** — avoids duplicating CRUD/collections/notifications/dashboard. (audit §1)
2. **Backend is the single streak source of truth**; Flutter `StreakService` is a façade. (reasoned in §1/§8)
3. **Category enum extended** (add 4 values) rather than lossy label mapping. (audit §1)
4. **Soft-delete via `isActive`**, completions preserved; `enabled` keeps meaning pause/resume. (audit §1/§5)
5. **Uncomplete = `/complete` with `status:'pending'`** — one write path, one idempotency key.
6. **Preference in SharedPreferences** (local, theme_provider pattern), default `both`. (audit §8)
7. **No schema-version field now** — all changes additive/default-safe.
8. **Linked workout is one-directional and explicit** — only the named workout auto-writes its linked routine completion; best-effort, non-fatal.
9. **Neutral (no-required-task) days are transparent** to the streak; strict reset otherwise, no grace day. (spec items 9/12)
10. **Upgrade `RoutineScreen` in place**, no 6th nav tab. (audit §7)
11. **Streak lookback is a fixed rolling 366-day window** for both current and best (named constant shared by routine + workout calcs); "best" = best within 366 days. (review Finding 3)
12. **Historical streak qualification is frozen by stored completion records**; pause/archive are forward-only. (review Finding 4)
13. **Present/future reads exclude `isActive===false` and `enabled===false`; historical reads are completion-record-driven** (`getDay` edited to branch on `today`). (review Finding 1)
14. **Workout-streak query uses a composite `status + completedDateKey` index** with the `status==='completed'` predicate kept. (review Finding 2)
15. **One per-day month aggregator** (`getMonthlyRoutineProgress`) behind both `/routines/month` (calendar) and `/streaks/summary` (streaks); no separate `/routines/range`. (review Finding 7)

---

## 15. Review response (revision 2)

Addresses every finding in `design-review.json` (`verdict: CHANGES_REQUESTED`). All HIGH and MEDIUM findings are resolved; NITs are addressed.

| # | Severity | Finding | Resolution | Where |
| --- | --- | --- | --- | --- |
| 1 | HIGH | `getDay`/calendar read path shows soft-deleted routines (no `isActive` filter) | **Addressed.** Added an explicit read-path filter rule: days `>= today` exclude `isActive===false` **and** `enabled===false`; historical days are driven by stored completion records (not a replay of current flags). `getDay` is now edited to take `todayKey` and branch, and is listed as an edited method. | §4.3 (new "Read-path filter rule"), §5.1, §13 |
| 2 | MEDIUM | Workout-streak query needs a composite index, not a single-field one | **Addressed.** Locked the exact query (`status=='completed'` + `completedDateKey` range) and added exactly one composite index `workouts: status ASC, completedDateKey ASC`; kept the `status` predicate with justification. | §3, §5.2, §13 |
| 3 | MEDIUM | Best-streak lookback ambiguous (366 days or all records) | **Addressed.** Locked a single fixed rolling **366-day** window via named constant `STREAK_LOOKBACK_DAYS`, shared by current+best and by routine+workout calcs; documented "best = best within 366 days." Removed the "or all records" alternative. | §5.1, §5.2 |
| 4 | MEDIUM | Paused-vs-historical rule not computable from stored data | **Addressed.** Reframed historical qualification to be driven by **stored completion records**, with pause/archive declared **forward-only** so a present-day toggle cannot rewrite past days. Added a historical-freeze unit test. True per-day pause history is explicitly out of scope. | §5.1 (step 1), §12, §11 |
| 5 | NIT | `endTime >= time` lexicographic compare looks accidental | **Addressed.** Noted that zero-padded `HH:mm` compares safely lexically, and specified a `toMinutes()` helper for an explicit numeric compare (unit-tested). | §4.1 |
| 6 | NIT | Monthly range single-field index claim misleading | **Addressed.** Stated no new `routineCompletions` index is needed (auto single-field + existing composite cover the `dateKey` range); removed the suggested entry. | §3 |
| 7 | NIT | Three endpoints can return month aggregates (drift risk) | **Addressed.** Removed `/routines/range`; exposed one calendar endpoint (`/routines/month`) and one streak endpoint (`/streaks/summary`), both backed by a single shared `getMonthlyRoutineProgress` aggregator and one `rangeBounds` helper. | §4.1, §4.2, §5.4 |

Unverified/wrong assumptions from the review doc are reconciled: the neutral-day/`computeStreak`-cursor interaction and the legacy-`isRoutineOnDate` back-compat branch are now listed as **required** unit-test cases (§11), not assumptions.

### Review response (revision 3)

Addresses the single finding in the latest `design-review.json` (`verdict: CHANGES_REQUESTED`, one MEDIUM, zero HIGH). Verified against the real code before changing (`backend/src/middleware/validate.ts` lets `ZodError` propagate; `backend/src/middleware/errorHandler.ts` maps `ZodError` → `res.status(422)`; `AppError` uses its own `statusCode`).

| # | Severity | Finding | Resolution | Where |
| --- | --- | --- | --- | --- |
| 1 | MEDIUM | §10 states Zod validation failures (incl. `superRefine`: `repeat==once` w/o `date`, `endTime<time`, `selected` w/o days) and the month/span check return **400**; the verified pipeline returns **422** for every `ZodError`. Wrong status would make implementers encode wrong integration-test assertions / client branches. | **Addressed.** Every "Zod validation fails → 400" row in §10 is now **422** (type/shape and `superRefine` both, since refinements raise `ZodError` inside `.parse()`). The 62-day span cap is split out and made explicit: it is a *cross-field runtime* guard enforced by a thrown `BadRequestError` (**400**) in the shared `rangeBounds` helper — deliberately **not** a Zod refine — because `year`/`month` are each individually schema-valid. Added an explicit status-code contract paragraph under the §10 table and matching 422/400 integration-test assertions in §11. 404 (`NotFoundError`) and 500 rows unchanged (correct). | §4.1 (`rangeBounds` 400 rationale), §10 (table rows + contract paragraph), §11 (status-code assertions) |

