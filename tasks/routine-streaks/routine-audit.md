# FitTrack — Routine / Goals / Calendar / Streaks Audit

Read-only audit for the "Daily Routine / Personal Goals / Progress Calendar / Strict Activity Streak" feature.
Scope: `frontend` (Flutter/Dart, branch `master`), `backend` (TypeScript/Express, branch `main`).

All paths below are repo-relative unless noted. No code was modified in this step.

---

## 0. Build & test commands (verified from manifests)

Backend (`backend/package.json`):
- Build / typecheck: `npm run build` (`tsc -p tsconfig.json`) or `npm run typecheck`.
- Tests: `npm test` (`vitest run`). Existing unit tests live in `backend/src/__tests__/` (e.g. `routineCalc.test.ts`, `streakCalc.test.ts`). Vitest is already a devDependency — reuse it, do not add a runner.
- Lint: `npm run lint`.

Frontend (Flutter):
- Analyze: `flutter analyze` (from `frontend/`).
- Tests: `flutter test` (existing tests under `frontend/test/`, e.g. `game_session_test.dart`).
- HARD CONSTRAINT: do **not** build an APK, do not run `flutter pub get` to add packages.

Pattern for verification in the plan: run `cd backend && npm test` for backend logic (streak/routine calc), `cd frontend && flutter analyze` + `flutter test` for Dart. Pure functions (streak/day-state calculators) should get Vitest/Dart unit tests — that is the real verification, not grep.

---

## 1. The existing "routine" IS the daily-goals concept (NOT workout templates)

This is the central extend-vs-sibling question, and the answer is clear: **EXTEND the existing routine feature. Do not add a sibling concept.**

Evidence — the existing `routine` is already a generic daily task/goal, not an exercise template:

- Model `frontend/lib/models/routine.dart`:
  - `Routine { id, name, description?, category, time (HH:mm), repeatDays (0=Sun..6=Sat), enabled, notificationEnabled, status }`.
  - `RoutineCategory` enum: `wakeUp, breakfast, lunch, dinner, snack, workout, water, sleep, study, work, custom`. Already generic (study/work/custom), not fitness-only.
  - `RoutineStatus { pending, completed, skipped }`.
  - `RoutineDay { dateKey, routines, total, completed, completionPercent }` — the "today's progress" DTO.
- Backend mirror `backend/src/models/domain.ts` → `ROUTINE_CATEGORIES` (same 11 wire values; `wake_up` snake_case).
- Firestore (per `docs/DATABASE.md`): `users/{uid}/routines/{routineId}` (definitions) and `users/{uid}/routineCompletions/{completionId}` (per-day records, doc id = `${dateKey}__${routineId}`, fields `dateKey, routineId, status, updatedAt`).

Workout *templates* are an entirely separate concept: `users/{uid}/workouts/{workoutId}` with `isTemplate` flag (see §4). So "routine" and "workout template" are already distinct. **The new feature maps almost 1:1 onto the existing routine model**; building a parallel "goals" collection would duplicate CRUD, completion tracking, Firestore collections, notifications, and the dashboard card.

### Gap analysis — fields the new spec needs that the existing `Routine` lacks

| New requirement | Exists today? | Action |
| --- | --- | --- |
| Title (`name`), description, category, time, repeat (daily/selected days via `repeatDays`), enable/disable (`enabled`), reminder toggle (`notificationEnabled`), completion w/ `pending/completed/skipped` | ✅ Yes | Reuse |
| Category set `Morning|Study|Workout|Health|Personal|Other` | ⚠️ Partial. Existing: `wakeUp,breakfast,lunch,dinner,snack,workout,water,sleep,study,work,custom`. No `Morning`/`Health`/`Personal`/`Other` exact labels. | Decide: either map new labels onto existing enum (e.g. Morning→wakeUp, Health→water/custom, Personal→custom, Other→custom) OR extend `ROUTINE_CATEGORIES` + `RoutineCategory`. Extending both enums (backend `domain.ts` + frontend `routine.dart`, kept in sync — comment in `domain.ts` mandates sync) is cleaner and matches the spec; it is additive and backward compatible. |
| `date` (one-off goal on a specific day) | ❌ No. Repeat model is weekday-only (`repeatDays`); no single-date / "Once" scheduling. | Add. Need a repeat mode `once` with a concrete `date` (dateKey), plus keep `daily` / selected-days. See §9 scheduling note. |
| Optional start time + end time | ⚠️ Only single `time` (HH:mm). | Add optional `endTime` (and treat `time` as start). |
| Priority `Low|Medium|High` | ❌ No | Add `priority` field (default medium). |
| `repeat: Once` | ❌ No (only daily / selected weekdays) | Add. |
| `isRequiredForStreak` (ON/OFF, default ON) | ❌ No | Add — core to Routine Streak (§5). |
| Pause / resume recurring | ⚠️ `enabled` bool exists and already excludes a routine from the "today" view (`routineService.getDay` filters `enabled !== false`). | Reuse `enabled` as pause/resume (semantically: paused = disabled). Confirm this does not break reminder scheduling (it already skips `!enabled`). |
| Completion timestamp | ⚠️ `routineCompletions` has `updatedAt` (server timestamp) but no explicit `completedAt`. | Add `completedAt` on the completion doc when status→completed (spec item 3 requires recording completion timestamp). |
| Linked workout (a routine item that is also "the workout") | ❌ No | Add optional `linkedWorkoutId` or a convention so completing the real workout auto-satisfies that routine item (spec item 13). See §4/§7. |

### CRITICAL BUG in current delete behavior (violates spec item 5)

`backend/src/services/routineService.ts` → `delete()` calls `routineRepository.deleteCompletionsForRoutine(uid, id)` (`backend/src/repositories/routineRepository.ts`), which **batch-deletes all completion history** for the routine. The spec explicitly requires: *"Deleting a goal must NOT destroy historical completion records; historical data stays available for analytics/streak."*

Required change: on delete, stop purging `routineCompletions`. Options: soft-delete the routine (add `deletedAt` / `archived` flag and exclude from active lists) OR hard-delete the definition doc but **keep** completion docs (they already carry `dateKey`/`routineId` and are self-sufficient for calendar/streak history). Recommended: soft-delete/archive so the calendar can still render the routine's name on historical days. This is a behavior change the plan must call out.

---

## 2. Existing streak logic — EXACT computation path

Grepped both repos for `streak|lastLogin|loginDate|appOpen|onAppStart|currentStreak`.

### Finding: the current app-wide streak is NOT login/app-open based today — it is the morning-game streak.

- Source of truth: `backend/src/utils/streakCalc.ts` → `computeStreak(successDateKeys, today)`. Counts consecutive distinct `dateKey`s (ending today or yesterday) that have a **successful morning-game** result. Pure, already unit-tested in `backend/src/__tests__/streakCalc.test.ts`.
- Fed by: `backend/src/services/gameService.ts` → `summary()` reads `users/{uid}/gameHistory` (`backend/src/repositories/gameRepository.ts`), filters `success === true`, maps `dateKey`, calls `computeStreak`.
- Exposed via: `GET /api/games/summary` (`backend/src/routes/game.routes.ts`, `app.ts` mounts `/api/games`).
- Frontend: `frontend/lib/features/games/data/game_repository.dart` (`GameSummary { streak, history }`) → `frontend/lib/features/games/providers/game_providers.dart` (`gameSummaryProvider`).
- Rendered on dashboard: `frontend/lib/features/home/presentation/dashboard_screen.dart` — `final streak = ref.watch(gameSummaryProvider).valueOrNull?.streak ?? 0;` → `_StreakBanner` (fire card). Comment literally says "Reuse the existing streak source of truth (morning-challenge streak)."
- Also on profile: `frontend/lib/features/profile/presentation/profile_screen.dart` — same `gameSummaryProvider.streak` shown as the "Day streak" stat.

### Implication for the "login increases streak is WRONG" requirement

Searched for `lastLogin`, `loginDate`, `appOpen`, `onAppStart` across both repos → **no matches**. The current productivity streak is therefore **not** literally computed from login/auth/app-open; it is keyed off real game-completion records. There is NO `lastLoginDate`-based streak to delete.

Two things the plan must still do to satisfy the spec:
1. The dashboard/profile "day streak" is currently the morning-game streak and is labeled generically ("day streak"). The spec wants **two new independent streaks (Routine + Workout)** surfaced on the dashboard via a user preference. Do NOT delete the game streak logic (it is the alarm/game feature — a preserved prior fix), but it is NOT one of the two new streaks. The dashboard banner must be rebuilt to show Routine/Workout streak(s) per the new preference (§8), and the morning-game streak stops being the dashboard's generic "day streak".
2. Explicitly ensure the two NEW streaks are computed only from `routineCompletions` (Routine) and completed `workouts` (Workout) — never from auth/login/app-open. Add a short note/test asserting this. The alarm/game splash flow must not feed the new streaks.

Note: `frontend/lib/core/routing/app_router.dart` has a splash screen and auth-aware redirects, and `frontend/lib/features/shell/home_shell.dart` runs startup side-effects (FCM register, reschedule reminders) in a post-frame callback — none of these touch any streak counter today. Leave them alone; just do not introduce streak writes there.

---

## 3. `computeStreak` is reusable for the new streaks

`backend/src/utils/streakCalc.ts::computeStreak(dateKeys, today)` is a generic "consecutive distinct day-keys ending today/yesterday" calculator. It can be reused directly for the **Workout Streak** (feed it the dateKeys of completed workouts) and as a base for the **Routine Streak** (feed it the dateKeys that satisfy the "all required routine tasks completed" rule). Recommend keeping it generic and adding a routine-specific "which days qualify" calculator alongside it, then reusing `computeStreak` for the consecutive-day count. Strict streak (no grace day) is already how `computeStreak` behaves — a gap resets it. This matches spec item 12 (no grace day).

---

## 4. Workout completion — the authoritative "completed" signal

The streak must key off actual completion, not page-open / add-exercise / workout-start.

Authoritative signal: a workout document transitions to `status === 'completed'`.
- Backend: `backend/src/services/workoutService.ts::transition(uid, id, to)`. When `to === 'completed'` it stamps `endedAt = serverTimestamp()`, computes `durationSeconds`, and runs PR detection. Status transitions are guarded by `canTransition` (`backend/src/utils/workoutCalc.ts`). Completed workouts are then immutable (`update()` throws `BadRequestError` if `status === 'completed'`).
- Route: `PATCH /api/workouts/:id/status` body `{ status }` (`backend/src/routes/workout.routes.ts` → `workoutController.setWorkoutStatus`).
- Frontend trigger: `frontend/lib/features/workout/presentation/active_workout_screen.dart::_finish()` → `active_workout_controller.dart::transition(WorkoutStatus.completed)` → `workout_repository.dart::setStatus()` → the PATCH. The "Finish" button is the only path to `completed`; "Start" sets `in_progress`; toggling a set (`toggleSetComplete`) only flips a set's `completed` flag and does NOT complete the workout. So "start" and "add set" correctly do NOT qualify — exactly what the spec wants.
- Firestore: `users/{uid}/workouts/{workoutId}` with `status`, `startedAt`, `endedAt`, `durationSeconds`. There is a declared composite index `status + startedAt DESC` (`firebase/firestore.indexes.json`).

### How to derive Workout-Streak day-keys
There is no `dateKey` field on workouts (unlike routineCompletions/waterLogs/foodLogs). The completion day must be derived from `endedAt` (the completion timestamp) converted to a local `yyyy-MM-dd`. Recommended approach (least intrusive, server-side, respects the strict-completion rule):
- In `workoutService.transition`, when `to === 'completed'`, also write a `completedDateKey` (local day of completion) onto the workout doc — OR compute streak from `endedAt`. Writing an explicit `completedDateKey` is cleaner for querying and mirrors the app's existing `dateKey` convention (`docs/DATABASE.md` "Date handling"). This is an additive field on completion only; it does not change workout internals or editing.
- A backend `GET /api/workouts/streak` (or extend the workout controller) can list completed workouts, map `completedDateKey`, and call `computeStreak`. Do NOT recompute on the client from page state.

Keep workout logging internals otherwise untouched (hard constraint).

---

## 5. Routine Streak — rule and data source

Default rule (spec item 9): a day counts only if **every required routine scheduled that day is completed**. "Required" = new `isRequiredForStreak` flag (default ON). Optional tasks must not break the streak.

Data available:
- Which routines are scheduled on a date: `backend/src/utils/routineCalc.ts::isRoutineOnDate(repeatDays, dateKey)` (empty `repeatDays` = every day). This is already used by `routineService.getDay`. For "Once" goals (new `date`), scheduling must also match a single dateKey — extend this calculator.
- Completion status per date: `routineCompletions` where `dateKey == X` (`routineRepository.listCompletionsByDate`). Doc id `${dateKey}__${routineId}`.
- `completionPercent(scheduled, completed)` already exists for the daily-progress card.

Routine-Streak computation (recommended, backend, pure + testable):
1. For each day in the window, determine the set of scheduled **required** routines (respecting `enabled`/pause, `repeatDays`, and new `once`+`date`).
2. The day "qualifies" if that set is non-empty AND all of them have a `completed` completion for that `dateKey`. (Decide the "no required tasks scheduled" case: a day with zero required tasks should be neutral/⚪, not a qualifying streak day and not a break — document this explicitly.)
3. Feed qualifying dateKeys to `computeStreak` for the consecutive count.

A subtle issue to call out in the plan: streak over a historical window needs each day's *scheduled-required* set reconstructed from the current routine definitions (repeatDays). Since routines can be edited/paused/deleted, document that streak is computed from the current definitions + stored completions (and that soft-deleting rather than purging completions, per §1, preserves history). This is a design decision the plan must state (one or two sentences of rationale), not leave open.

Linked workout (spec item 13): if a required routine item is a workout (new `linkedWorkoutId`), completing the actual workout should auto-write a `completed` routineCompletion for that routine on that day — only for the explicitly linked workout. Implement server-side in `workoutService.transition` (on `completed`) so the two streaks stay independent but the linkage is honored. Completing routine tasks must never write workout completions.

---

## 6. Monthly calendar + day detail — what the backend must expose

The calendar needs per-day aggregate states for a month: 🟢 full / 🟡 partial / 🔴 missed / ⚪ none / 🔥 streak day.

Current backend only has `GET /api/routines/today?date=` (single day via `routineService.getDay`). For the month view, add an endpoint that returns, for a month (or date range), per-day `{ dateKey, total, completed, state }` plus which days are routine-streak days. Build it on the existing primitives (`listCompletionsByDate` / a range query on `routineCompletions.dateKey`, `isRoutineOnDate`, `completionPercent`). A composite index already exists for `routineCompletions` (`dateKey ASC, routineId ASC`); a range query on `dateKey` is index-friendly.

Day detail (spec item 7) is essentially the existing `getDay(uid, dateKey)` for an arbitrary historical date — already supported (`/api/routines/today?date=yyyy-MM-dd`). Historical dates are viewable now; ensure the completion write path never silently overwrites a historical record without user action (today it uses `set(..., {merge:true})` keyed by `${dateKey}__${routineId}`, which is deterministic and safe — document that opening the calendar must not auto-write).

UI: build the calendar as compact selectable tiles (seat-selection concept). Follow existing widget conventions in `frontend/lib/widgets/` (e.g. `state_views.dart`, `gradient_card.dart`) and the routine screen layout. Reuse `AppColors`/`AppSpacing` (`frontend/lib/core/theme/`). Map states to theme colors (e.g. `AppColors.fireGradient` already exists for streak emphasis).

---

## 7. Dashboard integration (do not break water/reminders)

`frontend/lib/features/home/presentation/dashboard_screen.dart` renders, in order: header, `_StreakBanner`, deferred AI insight, Today's summary, Today's routine card (`TodayRoutineCard` from `widgets/dashboard_sections.dart`, fed by `routineTodayProvider`), Quick actions, **Water quick widget** (`WaterQuickWidget` — optimistic +250ml, see §10), **Reminders quick widget** (`RemindersQuickWidget`). The `RefreshIndicator.onRefresh` invalidates `nutritionDayProvider`, `waterDayProvider`, `routineTodayProvider`, `aiDailyInsightProvider`, `gameSummaryProvider`.

Changes needed, scoped tightly:
- Replace `_StreakBanner`'s data source: instead of the morning-game streak, render Routine/Workout streak card(s) per the new dashboard-streak preference (§8). The `_StreakBanner` widget UI (fire gradient card) can be reused/duplicated for each streak. "None" hides it.
- Add the new providers (routine streak, workout streak) and include them in `onRefresh` invalidation.
- Do NOT touch `WaterQuickWidget` / `RemindersQuickWidget` / nutrition / AI insight blocks.

Navigation to the new Routine/Goals page: the Routine screen is currently reachable only via Profile → "Daily routine" (`frontend/lib/features/profile/presentation/profile_screen.dart` → `RoutineScreen`). Routing uses `go_router` (`frontend/lib/core/routing/app_router.dart`) but the authenticated area is a single `/home` `HomeShell` with an `IndexedStack` of 5 tabs (`frontend/lib/features/shell/home_shell.dart`): Home, Workout, Nutrition, AI, Profile. The new Routine/Goals + calendar page can either (a) replace/upgrade the existing `RoutineScreen` reached from Profile and from a dashboard entry, or (b) be promoted. Recommended: upgrade the existing `RoutineScreen` in place (daily list + progress card already there) and add the monthly calendar + day-detail below it, plus a dashboard shortcut. Avoid adding a 6th bottom-nav tab unless the plan explicitly wants it (the shell hard-codes 5).

---

## 8. Dashboard-streak preference — where it lives

Spec item 14: user setting Routine / Workout / Both / None, default Both, persisted across restarts.

Two viable persistence patterns already in the repo:
- **SharedPreferences (local, synchronous)** — pattern in `frontend/lib/core/theme/theme_provider.dart` (`ThemeModeNotifier` reading/writing a single key via `sharedPreferencesProvider`, overridden in `main`). Simplest; matches "persisted across restarts"; no backend change. Recommended for this UI-only preference.
- **Profile doc (server, cross-device)** — `users/{uid}/profile/data` via `/api/profile` (`profile_providers.dart`). Heavier; only needed if cross-device sync is required (spec does not require it).

Recommendation: a `StateNotifierProvider<..., DashboardStreakPref>` backed by SharedPreferences, mirroring `theme_provider.dart` exactly (key e.g. `dashboard_streak_pref`, default `both`). Settings UI goes in `profile_screen.dart`'s "Settings" `Card` as a new `ListTile` with a dropdown/radio (same structure as the existing Theme dropdown). This is a local, reversible change.

---

## 9. Scheduling model: adding "Once" + specific date

Current scheduling is weekday-only (`repeatDays`, 0=Sun..6=Sat) with "empty = every day" (`routineCalc.isRoutineOnDate`). The spec adds `Repeat: Once` with a specific `date`, and `Daily`, and `Selected days`.

Recommended model (additive, backward compatible):
- Add `repeat` mode (`once|daily|selected`) and an optional `date` (dateKey) used when `repeat == once`. Keep `repeatDays` for `selected`; `daily` = all 7 / empty.
- Extend `isRoutineOnDate` (backend `routineCalc.ts`) to also return true when `repeat==once && date==dateKey`. Add a matching Dart helper if the frontend needs to compute scheduling locally (today it relies on the backend `getDay`).
- Reminder scheduling (`frontend/lib/features/notifications/reminder_scheduler.dart`) currently schedules weekly-repeating notifications from `repeatDays`. For a `once` goal, schedule a one-shot notification instead. This is the one notification-system touch point — reuse `NotificationService` (§11), do not build a new scheduler.

Keep existing routines (no `repeat` field) interpreted as before (default `daily`/`selected` from `repeatDays`) so no migration is needed.

---

## 10. Offline / optimistic update pattern to match

Reference implementation: `WaterQuickWidget` in `frontend/lib/features/home/presentation/widgets/dashboard_sections.dart`. It keeps a local `_optimisticAddMl`, applies it immediately on tap, invalidates `waterDayProvider` on success (reset counter), and rolls back + shows a retry SnackBar on failure. The routine completion checkbox today (`routine_screen.dart`) does a round-trip then `ref.invalidate(routineTodayProvider)` with no optimistic update — tapping a checkbox waits for the server. For the new completion UI, match the water optimistic pattern so the progress bar/percentage respond instantly on tap, then reconcile. Repositories return `Result<T>` (`Success`/`Err(Failure)`) — keep that convention (`frontend/lib/core/utils/result.dart`).

---

## 11. Notifications — reuse, do not rebuild

- `frontend/lib/core/services/notification_service.dart`: `NotificationService` singleton over `flutter_local_notifications`, timezone-aware, channels enum `NotifChannel { workout, meal, water, routine, pr, ai }`, `scheduleReminders`, `cancelAll`, `cancelRange`, `showNow`, and `stableNotificationBaseId(key)` for deterministic per-routine ids. Routine reminders already use channel `routine`.
- `frontend/lib/features/notifications/reminder_scheduler.dart`: `ReminderScheduler.rescheduleAll({routines, profile})` — idempotent cancel-all + reschedule from enabled routines (`enabled && notificationEnabled`) and lifestyle meals. `rescheduleReminders(ref)` wrapper (`notification_providers.dart`) reads `routinesProvider` + `profileProvider`. It is called after routine CRUD (`routine_editor_screen.dart`, `routine_screen.dart` delete) and on startup (`home_shell.dart`).
- For per-task reminders on the new fields (start time, "Once" date): extend `reminder_scheduler.dart` to honor the new `repeat`/`date`/`endTime` fields. Reuse `NotificationService`; do NOT introduce a new scheduler or package. Keep rescheduling idempotent so edits/deletes/pauses don't leave stale notifications.

---

## 12. Backend conventions new models/routes must follow

- Domain enums in `backend/src/models/domain.ts` (keep frontend `models/*.dart` in sync — mandated by the file's header comment).
- Validators: Zod schemas per endpoint in `backend/src/validators/*.ts` (see `routineValidators.ts`; `commonSchemas.dateKey`, `commonSchemas.timeOfDay` exist in `middleware/validate.js`). Reuse these for new fields.
- Routes: thin router per resource (`backend/src/routes/routine.routes.ts`), `router.use(authenticate)` first, `validate({...})` middleware, `asyncHandler`. Mounted in `backend/src/app.ts`.
- Auth: `backend/src/middleware/auth.ts` → `authenticate` + `requireUid(req)` derives UID from the verified Firebase ID token; **never** trust client UID. Every controller uses `const uid = requireUid(req);`. New routes MUST do the same (hard constraint: enforce `request.user.uid == resource.userId`). Since all data is namespaced `users/{uid}/...` and repos take `uid`, ownership is structural.
- Repositories: class with `getFirestore().collection('users').doc(uid).collection('<name>')`, `serverTimestamp()` on writes, `{id, ...data}` return shape (see `routineRepository.ts`, `gameRepository.ts`, `workoutRepository.ts`).
- Response envelope: `ok(res, data, status?)` → `{ success, data }` (`backend/src/utils/http.ts`).
- Firestore rules (`firebase/firestore.rules`): `users/{uid}/{document=**}` is already owner-only read/write at any depth — new subcollections (if any) inherit this automatically; no rule change needed for new per-user collections. If a new collection needs composite indexes (e.g. month range query on `routineCompletions.dateKey`, or workout `completedDateKey`), add them to `firebase/firestore.indexes.json` (do not touch project values/`google-services.json`).

---

## 13. Hard constraints reaffirmed (for the plan)

- No APK build. No dependency upgrades / new packages — reuse Riverpod, GoRouter, Firestore, Dio (`ApiClient`), `flutter_local_notifications`, `shared_preferences`.
- Do not modify `google-services.json`, `firebase_options.dart` project values, `applicationId`, signing config.
- No mock/fake data, no hardcoded UIDs. All UID from verified token.
- Do not touch unrelated features (workout logging internals, nutrition, water, AI, alarm/game) except the explicit integration points: (a) workout `transition→completed` adds `completedDateKey` + optional linked-routine auto-complete; (b) dashboard streak banner data source; (c) profile settings add the streak preference tile; (d) notification scheduler extension. Preserve prior fixes (auth/FCM in `home_shell`, AI history, alarm state machine, 10-min game / morning-challenge streak).

---

## 14. Suggested work decomposition (dependency-ordered, for the planner)

1. Backend model/validator extension: add `repeat`, `date`, `endTime`, `priority`, `isRequiredForStreak`, `linkedWorkoutId` to routine schema (`domain.ts`, `routineValidators.ts`); extend `routineCalc.isRoutineOnDate` for `once`. Unit-test calc. **Fix delete to preserve completion history** (soft-delete/archive). Add `completedAt` on completion.
2. Backend routine month/range endpoint + per-day state aggregation; add indexes if needed. Unit-test the day-state + routine-streak calculator (reuse `computeStreak`).
3. Backend workout completion → `completedDateKey` + linked-routine auto-complete; workout-streak endpoint reusing `computeStreak`. Unit-test.
4. Frontend model sync (`routine.dart`) + repository methods for new fields/endpoints.
5. Frontend Routine/Goals page upgrade: new editor fields, optimistic completion (water pattern), daily-progress card (already present), monthly calendar tiles + day detail.
6. Frontend streak providers + dashboard banner rework + profile "Dashboard streak" preference (SharedPreferences, theme_provider pattern).
7. Frontend reminder-scheduler extension for new fields; verify idempotent reschedule.
8. Verification: `cd backend && npm test` and `npm run build`; `cd frontend && flutter analyze && flutter test`.

(These are candidate FEAT boundaries; features 1–3 are backend and independent-ish but ordered by data dependency; 4–7 depend on backend contracts.)
