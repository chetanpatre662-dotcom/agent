# Three-bug fix review — FitTrack AI

The change set fixes three independent functional bugs that the audit proved: the AI chat-history 500, the morning alarm dying after ~2 rings, and the morning game completing in seconds. Backend fixes the history read-path at the query that required a missing composite index; frontend rewrites the native alarm state machine to a date-keyed, indefinitely-retrying, state-guarded loop and gates the morning game behind a wall-clock 10-minute floor using only non-math games. The work is spread across tracked edits (`AlarmModels.kt`, `game_registry.dart`, `morning_challenge_screen.dart`, `game_registry_test.dart`, `aiConversationRepository.ts`) and a larger set of untracked new files that carry the bulk of the alarm/game logic and all the new tests — these were read directly from disk since `git diff` does not surface untracked files.

**Watch for:** verification evidence from the implementer was not found in the task directory or in git (the only backend commit is an unrelated FEAT-002), so a single narrow spot-check was run on the backend history test file (8/8 green) to confirm the root-cause fix; the remaining suites rely on reading the code and tests, which are meaningful and consistent. One stale doc comment ("2-minute re-ring" in `AlarmActivity.kt`) and the optional defense-in-depth composite index were not addressed — both non-blocking.

**Verdict**: APPROVED

## High-level view

The AI history 500 is fixed at its proven root cause. `listConversations` no longer issues the composite `where('role','==','user').orderBy('createdAt')` query that needed an index Firestore never had; it reads the first message with `orderBy('createdAt').limit(1)` (covered by the existing index) and derives the title through a pure, null-safe helper. The fix is a real query change, not a catch-and-return-empty, and ownership stays scoped to `users/{uid}` with the UID coming from the verified token.

The alarm is now a persisted, date-keyed state machine whose critical race guard runs at the native receiver execution point, not just in UI. `AlarmReceiver.onReceive` checks completed/gameRunning before ringing and re-arms the next 5-minute re-ring from inside the receiver every cycle, so retries continue with no attempt cap. Opening the game cancels the pending re-ring and sets GAME_RUNNING; completing it cancels all retries and marks COMPLETED idempotently. Sessions are keyed by `yyyy-MM-dd` so a new day resets stale state, and the primary-fire path still reschedules repeating alarms (boot behavior preserved).

The morning game cannot complete before a hard 10-minute floor computed as a wall-clock delta from a `startedAt` persisted in SharedPreferences keyed by alarm id + date, so backgrounding or activity recreation cannot shorten it or auto-complete the session. Finishing all planned stages early appends further non-math rounds until the floor elapses. The morning pool excludes the two arithmetic games (`numberSequence`, `logicPuzzle`) via an allow-list enforced in `GameRegistry.pickRandom`, while those games stay registered for non-morning use.

<details>
<summary>Issues (3)</summary>

1. **Missing verification evidence** — the implementer's recorded test-run results were not found in the task directory or git; mitigated by a narrow spot-check of the backend history suite (8/8 pass). Non-blocking given the code and tests read as correct, but future passes should capture the full-suite output.
2. **Stale re-ring comment** — `AlarmActivity.kt` and `AlarmModels.kt` still reference a "2-minute re-ring" / `challengeTimeoutSeconds` 1200 default in comments while the code uses the 5-minute `GAME_OPEN_DEADLINE_MILLIS` constant. Cosmetic; update the comments to avoid confusion.
3. **Optional composite index not added** — the audit's defense-in-depth `messages (role, createdAt)` index was not added to `firebase/firestore.indexes.json`. Not required (the query no longer needs it), but it was listed as a nice-to-have.

</details>

<details>
<summary>Details</summary>

### AI history: the composite-index query is gone, not masked

`backend/src/repositories/aiConversationRepository.ts` `listConversations` previously issued `where('role','==','user').orderBy('createdAt','asc').limit(1)` per conversation — the composite-index-requiring query the audit proved was throwing `FAILED_PRECONDITION`, which fell through the error handler to a generic 500 and surfaced client-side as `ServerFailure(internal_error ...)`. The fix drops the `where('role')` filter and reads `orderBy('createdAt','asc').limit(1)` only, which the existing `messages (createdAt ASC)` index covers. The title is derived by the pure `titleFromFirstMessage`, which only uses a first doc whose `role === 'user'` and otherwise falls back to the generic label — matching the invariant that `aiController.chat` persists the user turn first.

Ownership is unchanged and correct: `conversations(uid)` scopes every read to `users/{uid}/aiConversations`, and the route derives the UID from the verified token (the auth-gate test asserts an unauthenticated request gets 401 with `UNAUTHORIZED`, so another user's data can never be returned). The mappers are extracted as pure, exported functions (`mapConversationSummary`, `mapMessage`, `toIso`) with defensive null/type handling — non-string roles/text, numeric garbage, `NaN` counts, and missing/odd timestamps all default safely rather than throwing, so a single malformed document cannot 500 the endpoint. This is the audit's prescribed fix implemented faithfully, with no catch-and-return-empty anywhere in the path.

The backend test file (`aiHistory.test.ts`) covers user-with-history, empty history, malformed doc, invalid/missing timestamps, the title-derivation rule, defensive message mapping, the success-envelope shape, and the 401 auth gate via a real `express` + `fetch` server. Running this one file as a spot-check: 8/8 pass. The frontend `ai_history_test.dart` confirms the repo hits `/api/ai/conversations` and `/api/ai/conversations/:id/messages`, parses populated and empty lists, and tolerates null `updatedAt`, empty `title` (falls back to `Conversation`), missing `messageCount`, and garbage field types without crashing. (confirmed)

The defense-in-depth composite index the audit mentioned as optional was not added to `firestore.indexes.json`. Since the query no longer needs any composite index, this does not block. (confirmed)

### Alarm: execution-point guard + indefinite re-arm

The core defect — the re-ring was armed once and never re-armed, dying after ~2 rings — is fixed inside `AlarmReceiver.onReceive`. On a re-ring the receiver applies the race rule at the AlarmManager execution point:

```
if sessionDate is stale (previous day) -> return
if isCompleted(alarm)                   -> return
if isGameRunning(alarm)                 -> return
else: ring; set 5-min deadline; scheduleRering() again  // re-arm
```

Because `scheduleRering` is called from inside the receiver on every cycle, the loop continues with no attempt cap, which `alarm_session_state_test.dart` proves by running 20 dismiss→deadline→refire cycles and asserting attempts keep incrementing. The deadline is the 5-minute `MorningSession.GAME_OPEN_DEADLINE_MILLIS` constant, not the old 20-minute `challengeTimeoutSeconds`. The re-ring reuses the single `(alarm, rering)` PendingIntent request code (`alarmId*10+1`) with `FLAG_UPDATE_CURRENT | FLAG_IMMUTABLE`, so retries replace rather than accumulate, and `setAlarmClock` keeps it exempt from Doze.

Suppression is enforced where it matters: `MainActivity.gameStarted` cancels the pending re-ring, stops sound/notification, and sets `GAME_RUNNING` (so the receiver's `isGameRunning` guard short-circuits any in-flight re-ring); `gameCompleted` cancels all retries, stops everything, sets `COMPLETED`, and is idempotent against duplicate callbacks. State is persisted in `AlarmStore` with `sessionDate`, `gameOpenDeadlineMillis`, and `gameStartMillis`, and the primary-fire path stamps a fresh `sessionDate`/clears session fields, so a new day starts clean and stale previous-day `COMPLETED`/`GAME_RUNNING` cannot leak. The primary fire still calls `AlarmScheduler.schedule` first for repeating alarms, preserving the next-occurrence scheduling, and `BootReceiver`/`rescheduleAll` is untouched.

The pure Dart `AlarmSessionState` mirrors these transitions (`idle → alarmRinging → alarmDismissed → waitingForGame → gameRunning → completed`) and its test suite covers first fire, 5-minute deadline on dismiss, refire when the game isn't opened, the no-cap retry loop, game-open cancelling the retry, no-ring-while-running, completion cancelling all retries, idempotent completion, and next-day reset. Native receiver re-arm and the execution-point guard cannot run under `flutter test`; the code carries documentation of the intended on-device behavior but the manual-verification steps live in the plan rather than inline — acceptable per the criteria, which only require the Dart state-model tests plus documented manual steps for native-only logic. (confirmed)

A stale comment remains: `AlarmActivity.kt`'s class doc still says "schedules a 2-minute re-ring" and `AlarmModels.kt` still annotates `challengeTimeoutSeconds` as "default 1200 (20 minutes)". The live code uses the 5-minute constant everywhere, so this is cosmetic only. (confirmed)

### Game: wall-clock 10-minute floor and non-math pool

`GameSession` computes `elapsed` as `now.difference(startedAt)` and gates completion on `canComplete = allChallengesFinished && elapsed >= 10 minutes`. In `morning_challenge_screen.dart`, `startedAt` is resolved from SharedPreferences keyed by `alarmId + yyyy-MM-dd` and persisted on first entry, so backgrounding or activity recreation recomputes elapsed from real time and never resets the clock. When all planned stages finish before the floor, `_onStageCompleted` calls `_appendRound()` (pulling another game from the non-math pool, avoiding an immediate repeat) and sets `_awaitingMinimum`, keeping the session active; a 1-second ticker only refreshes the UI and re-checks the gate, and `_completeSession` has a hard guard that re-appends a round if invoked before the floor. `reportGameCompleted` fires only once `canComplete` is true, and the persisted `startedAt` is cleared on genuine completion so the next morning starts fresh. There is no auto-complete on pause/background — completion only happens on a successful final-round result after the floor. (confirmed)

`game_session_test.dart` asserts no completion at 30s/1m/3m/5m/9m59s, that finishing all challenges early does not bypass the minimum, that completion works at exactly 10m and later, that an unfinished-challenge state blocks completion past 10m, and that a session rebuilt from the same persisted `startedAt` reports real elapsed time (the lifecycle-survival case). The non-math constraint is enforced in `GameRegistry.pickRandom`: the pool is drawn from `morningPool` (eight cognitive/reaction/memory games) and any requested ids are filtered down to it, so a math game can never be selected even if explicitly preferred — `game_registry_test.dart` verifies this by requesting only `numberSequence`/`logicPuzzle` across 100 draws and asserting the result is always in the non-math pool. `numberSequence` and `logicPuzzle` remain registered in `GameRegistry.build` for non-morning use, so no game was deleted or duplicated. (confirmed)

### Hard constraints

The frontend diff touches only `AlarmModels.kt`, `game_registry.dart`, `morning_challenge_screen.dart`, and `game_registry_test.dart` among tracked files; the backend diff touches only `aiConversationRepository.ts` (plus a `node_modules/.vite` cache artifact that is build noise, not a source change). No `google-services.json`, `firebase_options.dart`, `applicationId`, signing config, `pubspec.yaml`, or `package.json` change appears. No auth check was removed, no response faked, and the math games were excluded rather than removed, so no duplicate alarm or game system was introduced. The `node_modules/.vite/vitest/results.json` churn should ideally not be committed but is harmless. (confirmed)

</details>

<details>
<summary>File map</summary>

- `backend/src/repositories/aiConversationRepository.ts` — removed composite-index title query; added pure `titleFromFirstMessage`/`mapConversationSummary`/`mapMessage` helpers.
- `backend/src/__tests__/aiHistory.test.ts` (new) — history mapping + auth-gate regression tests.
- `frontend/.../alarm/AlarmModels.kt` — new session states, `MorningSession` helpers/constant, session runtime fields.
- `frontend/.../alarm/AlarmReceiver.kt` (untracked) — execution-point race guard + indefinite re-arm.
- `frontend/.../alarm/AlarmScheduler.kt` (untracked) — 5-minute re-ring via `setAlarmClock`, reused PendingIntent slot.
- `frontend/.../alarm/AlarmStore.kt`, `AlarmActivity.kt`, `MainActivity.kt` (untracked) — persisted mutate helper, dismiss→deadline wiring, gameStarted/gameCompleted channel handlers.
- `frontend/lib/features/alarm/alarm_session_state.dart` (untracked) + `test/alarm_session_state_test.dart` — pure Dart state model + tests.
- `frontend/lib/features/games/game_session.dart` (untracked) + `test/game_session_test.dart` — 10-minute wall-clock gate + tests.
- `frontend/lib/features/games/game_registry.dart` — non-math morning allow-list in `pickRandom`.
- `frontend/lib/features/games/presentation/morning_challenge_screen.dart` — persisted `startedAt`, append-round loop, floor-gated completion.
- `frontend/test/game_registry_test.dart`, `test/ai_history_test.dart` (untracked) — non-math pool + history parsing tests.

Full diff: `git diff master` (frontend) / `git diff main` (backend); untracked files read directly from disk.

</details>
