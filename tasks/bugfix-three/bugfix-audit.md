# FitTrack AI — Three-Bug Audit (READ-ONLY)

Scope: prove the root cause of three independent functional bugs from code actually read in this repo, and record the design targets the fix must implement. No source was modified in this step.

Repos:
- Frontend (Flutter/Dart): `c:\Users\cheta\OneDrive\Desktop\fit\frontend` (branch `master`)
- Backend (Node/TS): `c:\Users\cheta\OneDrive\Desktop\fit\backend` (branch `main`)

Build/test commands discovered:
- Backend: `npm run build` (tsc), `npm test` (vitest run) in `backend/`. Tests live in `backend/src/__tests__/*.test.ts` and mock repositories with `vi.mock` (they do NOT hit real Firestore — see `aiConversationService.test.ts`, `middleware.test.ts`).
- Frontend: `flutter analyze` and `flutter test` in `frontend/`. Tests live in `frontend/test/*_test.dart` and use `flutter_test` + `mocktail`; API is faked by implementing `ApiClient` (see `test/ai_history_test.dart`).
- Native Kotlin alarm logic cannot run under `flutter test`; it requires manual on-device verification (documented per item).

---

## BUG 1 — AI Chat History returns 500 "internal_error / an unexpected error occurred"

### Proven root cause
A **Firestore composite-index-requiring query** inside the conversation-list read path throws `FAILED_PRECONDITION` ("The query requires an index"). That throw is not an `AppError`, so the centralized error handler falls through to its generic branch and returns HTTP 500 with `{ code: 'INTERNAL_ERROR', message: 'An unexpected error occurred' }`. The Flutter client maps any `status >= 500` to `ServerFailure`, producing the exact user-visible string.

### Exact file / line / mechanism
- Throw origin: `backend/src/repositories/aiConversationRepository.ts`, `listConversations()`, the per-conversation "title" query:
  ```ts
  const firstSnap = await doc.ref
    .collection('messages')
    .where('role', '==', 'user')     // equality filter
    .orderBy('createdAt', 'asc')     // + orderBy on a DIFFERENT field
    .limit(1)
    .get();
  ```
  A single equality filter on `role` combined with `orderBy('createdAt')` requires a **composite** index on the `messages` collection `(role ASC, createdAt ASC)` with `COLLECTION` scope.
- Index inventory: `firebase/firestore.indexes.json` contains only:
  - `aiConversations` `(updatedAt DESC)` — covers the top-level `conversations().orderBy('updatedAt','desc').limit(50)` query (fine).
  - `messages` `COLLECTION_GROUP` `(createdAt ASC)` — covers an `orderBy('createdAt')`-only query, but **does NOT** cover `where('role') + orderBy('createdAt')`.
  There is no `messages (role, createdAt)` composite index. → Firestore rejects the query.
- Error normalization: `backend/src/middleware/errorHandler.ts` — the final branch:
  ```ts
  logger.error({ err }, 'Unhandled error');
  res.status(500).json({ success:false, error:{ code:'INTERNAL_ERROR', message:'An unexpected error occurred', ... }});
  ```
  (Note: on-wire the client lowercases/relabels it as `internal_error`; see mapping below.)
- Client mapping: `frontend/lib/core/services/api_client.dart` line ~94 `if (status >= 500) return ServerFailure(message, code: code);`. `ServerFailure.toString()` → `"ServerFailure(code: internal_error, message: an unexpected error occurred)"`, shown by `ErrorView(message: e.toString())` in `frontend/lib/features/ai/presentation/ai_chat_history_screen.dart`.

### Why the other candidates are NOT the cause (ruled out from code)
- Auth/UID extraction is correct: `ai.routes.ts` applies `authenticate`; controller uses `requireUid(req)` and never trusts client UID. `middleware/auth.ts` derives UID from the verified token. (A 401 would show `AuthFailure`, not `ServerFailure`.)
- Timestamp serialization is safe: `aiConversationRepository.toIso()` null-guards and handles `Timestamp | string | Date | null`; a null `createdAt`/`updatedAt` returns `null`, not a throw.
- Frontend parsing is null-safe: `models/ai_conversation.dart` `fromJson` tolerates null `updatedAt`, empty `title`, missing `messageCount`. `ai_repository.dart` `_wrap` catches and returns `Err`. So the frontend cannot be the 500 source; `internal_error` is unambiguously a backend 500.
- The top-level `aiConversations` query and `getMessages` (`orderBy('createdAt','asc')` only) each have a covering index and do not throw.

### Proposed minimal fix (root cause, not a mask)
In `aiConversationRepository.listConversations()`, make the title query **index-free** instead of relying on an index that this workflow cannot deploy:
- Drop the composite `where('role','==','user') + orderBy('createdAt')`. The conversation's first message is always the user turn (`aiController.chat` persists `'user'` then `'assistant'`), so fetch the first message by `orderBy('createdAt','asc').limit(1)` only (covered by the existing `messages (createdAt ASC)` index), and derive the title from it; if that first doc happens to be non-user, fall back to the generic label. This removes the index dependency entirely.
- Defense-in-depth: ALSO add the composite index `messages (role ASC, createdAt ASC)` COLLECTION scope to `firebase/firestore.indexes.json` so any future `role`-filtered + ordered query is covered. (indexes.json is allowed to change; it is NOT google-services.json / firebase_options.dart / project config.)
- Do NOT catch the exception and return empty history. The query is corrected so it no longer throws.
- Keep ownership scoping unchanged (`users/{uid}/aiConversations/...`, UID from verified token only).

### Regression tests to add (backend vitest, `backend/src/__tests__/`)
Mock `aiConversationRepository` Firestore access (as `aiConversationService.test.ts` already does) or unit-test a extracted pure mapper. Cover:
1. authenticated user WITH history → conversations mapped (title/updatedAt/messageCount).
2. authenticated user with EMPTY history → `[]`, no throw.
3. malformed/partial conversation doc (missing fields) → mapped with safe fallbacks, no throw.
4. invalid/missing `createdAt`/`updatedAt` timestamp → `toIso` returns null, no throw.
5. backend auth failure path → 401 UNAUTHORIZED (middleware test already models this; extend for the AI route shape).
6. successful response envelope shape `{ success:true, data:{ conversations:[...] } }`.

### Frontend tests to add/extend (`frontend/test/`)
7. history provider/repository parsing of a response with null/missing fields (extend `ai_history_test.dart`): null `updatedAt`, empty `title`, absent `messageCount` → safe defaults; empty list and non-empty list both parse.

---

## BUG 2 — Morning alarm rings only ~2 times then stops

### Proven root cause
The challenge re-ring is **armed exactly once** (at manual dismissal) and is **never re-armed when the re-ring itself fires**. The receiver that handles the re-ring broadcast rings but schedules no successor, so after the single armed re-ring the loop dies. Compounding this, there is **no persistent-state guard at the receiver execution point**, so the few rings that do occur ignore game-running/completed state (violating the race rule), and the deadline is 20 min (`challengeTimeoutSeconds` default 1200), not the required 5-min cycle.

### Exact file / line / mechanism
- Single-arm scheduling: `frontend/android/app/src/main/kotlin/.../alarm/AlarmActivity.kt`, `onTurnOff()` → `AlarmScheduler.scheduleRering(this, alarm)` is called only from the UI dismiss path.
- Re-ring handler does NOT re-arm: `.../alarm/AlarmReceiver.kt`, `onReceive()` — on `isRering=true` it sets state `RERINGING`, starts sound, shows the notification, and returns. It never calls `scheduleRering` again. The repeating-alarm reschedule (`if (!isRering && alarm.repeatDays.isNotEmpty()) schedule(...)`) is explicitly gated to NON-rering fires. → the chain is: fire(1) → user turns off → one re-ring armed → re-ring fires(2) → **nothing re-armed**. That is the "~2 rings then stops."
- No execution-point state check: `AlarmReceiver.onReceive` rings unconditionally. There is no `if (completed) return; else if (gameRunning) return;` guard, so a stale re-ring can ring even while the game is running or after completion (the race the spec forbids).
- Deadline/duration mismatch: `.../alarm/AlarmScheduler.kt`, `scheduleRering()` uses `alarm.challengeTimeoutSeconds * 1000L`; the default is `1200` (20 min) per `AlarmModels.kt` and `models/alarm.dart`. Spec requires a **5-minute** game-open deadline cycle.
- PendingIntent request codes: `AlarmScheduler.firePendingIntent` uses `alarmId*10 + (isRering?1:0)` with `FLAG_UPDATE_CURRENT`. Main vs re-ring don't collide, but because there is only ever one "rering" slot, a correct repeating loop must deliberately reuse/replace that slot each cycle (idempotent, non-accumulating) — acceptable, but the receiver must re-arm it.
- Doze: `scheduleRering` uses `setAlarmClock` (exempt from Doze, user-visible) — acceptable; spec also allows `setExactAndAllowWhileIdle`. The re-ring not being re-armed is the dominant defect, not Doze.
- State keys: `AlarmModels.kt` state machine (`SCHEDULED → RINGING → MANUALLY_DISMISSED → CHALLENGE_ACTIVE → GAME_STARTED → GAME_COMPLETED → COMPLETED`, plus `TIMEOUT/RERINGING/SNOOZED`) exists but is **not keyed by morning-session date**, so previous-day `COMPLETED`/`GAME_STARTED` state can leak into the next day. `AlarmStore` persists per-alarm `state` + `lastRingAtMillis` only.

### Flow confirmation (files read)
`AlarmActivity.onTurnOff` → state `CHALLENGE_ACTIVE` + `scheduleRering` + launch `MainActivity` with action `OPEN_MORNING_CHALLENGE`. `MainActivity` method channel: `gameStarted` → state `GAME_STARTED`; `gameCompleted` → `cancelRering` + cancel notification + stop sound + state `COMPLETED`. Flutter side: `home_shell.dart` routes to `MorningChallengeScreen`; `morning_challenge_screen.dart` reports `reportGameStarted`/`reportGameCompleted` via `alarm_service.dart`. `BootReceiver` reschedules enabled alarms on boot (preserve this).

### Design target the fix MUST implement (persisted, keyed by morning-session date)
State machine: `idle → alarmRinging → alarmDismissed → waitingForGame → gameRunning → completed`.
- Triggered → ringing. Dismiss → `alarmDismissed` → start a **5-minute** game-open deadline → `waitingForGame`.
- If game NOT opened within 5 min → ring again. On next dismiss → a NEW 5-min deadline. Repeat **indefinitely** — no 2/3/5-attempt cap, no artificial limit.
- Open game → `gameRunning` → cancel any pending retry alarm; while `gameRunning` the alarm must NOT ring.
- Game genuinely completed → `completed` (MORNING_ROUTINE_COMPLETED) → cancel all pending retries, stop notifications, clear deadline/retry state, prevent duplicate completion callbacks, no more rings that morning.
- Next day = fresh session keyed by date; stale previous-day state must NOT leak.

CRITICAL race rule (native, at the AlarmManager/Receiver execution point — NOT only UI):
```
onReceive():
  if morningRoutineCompleted(session): return   // do nothing
  else if gameRunning(session):        return   // do nothing
  else: ring; then RE-ARM the next 5-min re-ring
```
- Re-arm the re-ring from inside the receiver each cycle so the loop continues indefinitely.
- Make the receiver idempotent; keep PendingIntent request codes unique per (alarm, rering) and reuse/replace (don't accumulate duplicates).
- Use `setExactAndAllowWhileIdle` (or keep `setAlarmClock`) for the retry so Doze doesn't drop it.
- Persist: session id/date, alarm state, game-open deadline timestamp, game-running flag, game start time, completion flag (in `AlarmStore`/SharedPreferences).
- Preserve existing `BootReceiver` reschedule behavior.

### Tests
- Dart-side alarm **state model** (new pure Dart class mirrorable in `flutter test`): first fire; dismiss starts 5-min deadline; no game after 5 min → fires again; retry continues on 3rd/4th/5th with NO cap; game opens → retry cancelled; `gameRunning` → no ring; game completes → all retries cancelled; completed morning → no duplicate; next-day session resets.
- Native Kotlin (receiver re-arm + execution-point guard): document **manual on-device verification** steps (cannot unit-test here).

---

## BUG 3 — Morning game completes before 10 minutes; must be non-math and ≥10 min

### Proven root cause
Completion is decided **purely by finishing the configured number of game stages** — there is NO minimum-duration gate anywhere. The session ends as soon as the last stage reports success (seconds to a few minutes). Separately, two **math-based games are wired into the morning pool**, violating the non-math requirement.

### Exact file / line / mechanism
- Early completion: `frontend/lib/features/games/presentation/morning_challenge_screen.dart`, `_onStageCompleted()`:
  ```dart
  final isFinalStage = _stageIndex >= _session.stageCount - 1;
  ...
  // Final stage done → complete the whole challenge.
  await _completeSession(result);
  ```
  `_completeSession` immediately calls `reportGameCompleted` + pops. There is no `elapsedTime >= 10 minutes` check. `ChallengeSession` (`challenge_session.dart`) is a pure stage planner (default 3 stages) and its own doc comment states it "naturally lasts ~2–5 minutes" — well under 10 minutes.
- No session clock: `GameResult` (`game.dart`) has `startedAt`/`completedAt`/`durationSeconds`, but nothing gates completion on duration; `durationSeconds` is only recorded for history.
- Math games wired: `frontend/lib/features/games/game_registry.dart` registers all `GameId.values`, and `pickRandom` with empty `preferred` draws from `GameId.values`, which includes:
  - `GameId.numberSequence` → `games/number_sequence_game.dart`: arithmetic sequences (`start + step*i`), its own header comment says "arithmetic sequence". **Math.**
  - `GameId.logicPuzzle` → `games/logic_puzzle_game.dart`: "multiple of n", doubling sequence (`start*2, start*4, start*8`), "largest number". **Arithmetic.**
  The morning challenge (`MorningChallengeScreen`) builds `ChallengeSession.build(preferred: widget.preferredGames)`; when an alarm has no `preferredGames` (default `const []`), the pool is all games → a math game can be selected for the morning challenge.

### Non-math games already available (reuse, no new deps)
`memoryMatch`, `memoryMatrix`, `patternMemory`, `reactionTest`, `quickTap`, `colorShapeMemory`, `wordScramble`, `simon` are cognitive/reaction/memory/attention games. Memory Matrix was added recently and is a good fit.

### Design target the fix MUST implement
- **Non-math only**: exclude `numberSequence` and `logicPuzzle` from the morning-challenge pool (define a non-math allow-list / morning pool used by `ChallengeSession.build` and `GameRegistry.pickRandom` for the morning challenge). Do not delete the games wholesale unless simplest; at minimum they must never be selectable as the morning game. Confirm no arithmetic game remains wired.
- **GameSession model** (new, testable): `startedAt`, `currentRound`, `score`, `elapsedTime`, `minimumDuration` (10 min), `isRunning`, `isCompleted`.
- **Hard 10-minute gate**: `canComplete = allChallengesFinished && elapsedTime >= 10 minutes`, where `elapsedTime` is a **wall-clock delta** from a persisted `startedAt` (`DateTime.now().difference(startedAt)`), NOT a UI countdown. If the user finishes all rounds before 10 min, the session stays active and keeps generating further rounds/stages (reusing the non-math games) until 10 minutes have elapsed, then completes when the final condition is met.
- **Anti-exploit**: backgrounding must not auto-complete; never mark completed on pause/background; persist `startedAt` so elapsed is computed from real time even across activity recreation/backgrounding; `gameRunning` state must survive lifecycle transitions.
- **Signal wiring**: report `gameRunning` to the alarm state machine on first genuine interaction (`reportGameStarted`), and `gameCompleted` ONLY when `canComplete` is true (`reportGameCompleted`).

### Tests (`frontend/test/`)
- Game session: cannot complete before 10 min; finishing all challenges early does NOT bypass the minimum (stays active, keeps generating rounds); elapsed computed correctly from `startedAt`; completion after 10 min works when final condition met; state survives lifecycle transitions (compute elapsed from persisted `startedAt`, not a ticking timer).
- Non-math pool: assert the morning pool excludes `numberSequence` and `logicPuzzle` (extend `game_registry_test.dart` / `challenge_session_test.dart`).

---

## Hard constraints carried into every fix step
- Do NOT modify: `google-services.json`, `firebase_options.dart` project values, `applicationId`, signing config, Firebase project config. (Editing `firebase/firestore.indexes.json` is allowed — it is not project config.)
- Do NOT upgrade deps or add packages (`frontend/pubspec.yaml`, `backend/package.json`).
- Do NOT remove auth checks, hardcode UIDs, fake success responses, or hide backend exceptions behind empty-state UI. BUG 1 must be fixed at the query, not masked.
- Do NOT touch unrelated features (workout, nutrition, routine, water, dashboard, profile, progress, auth/FCM).
- Reuse existing services/providers/architecture — no duplicate alarm systems or game engines. Reuse existing games for the non-math morning pool.
