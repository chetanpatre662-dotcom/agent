# FitTrack AI — Three-Bug Fix Report

Scope: fix three functional bugs (AI chat-history 500, morning alarm dying after ~2 rings, morning game finishing before 10 minutes). No APK was built. Nothing was pushed.

Commits:
- Backend (branch `main`): `30f5380` — `fix: resolve AI chat history 500 by dropping composite-index query`
- Frontend (branch `master`): `e34bf75` — `fix: reliable morning alarm re-ring loop and 10-minute non-math game`

---

## 1. AI chat history — root cause

Opening chat history produced `ServerFailure (code: internal_error, message: an unexpected error occurred)`. The root cause was in `backend/src/repositories/aiConversationRepository.ts` → `listConversations`. For each conversation it issued a per-conversation title query that combined an equality filter with an ordering on a different field:

```
.collection('messages').where('role','==','user').orderBy('createdAt','asc').limit(1)
```

A `where(field A) + orderBy(field B)` query requires a Firestore **composite index** (`role ASC, createdAt ASC`) that the project never defined. Firestore rejected the query with `FAILED_PRECONDITION`, which propagated through the backend error handler as a generic 500 (`internal_error`) and surfaced on the client as the `ServerFailure`. The failure was in the server query, not in the UI — the client was faithfully reporting a real backend 500.

## 2. AI chat history — fix

The fix removes the composite-index requirement at its source rather than masking it:

- `listConversations` now reads the first message with `orderBy('createdAt','asc').limit(1)` only — no `where('role')` filter — which is covered by the existing single-field `messages (createdAt ASC)` index, so no composite index is needed.
- The title is derived by a pure, null-safe helper `titleFromFirstMessage`: it produces a message-based title only when the first doc is a `user` turn (which is the invariant, since `aiController.chat` persists the user turn before the assistant turn) and otherwise falls back to a generic label.
- Defensive pure mappers were extracted and exported: `mapConversationSummary`, `mapMessage`, and `toIso`. Non-string roles/text default safely, non-finite / negative message counts clamp to `0`, and timestamps (Firestore `Timestamp`, ISO string, `Date`, or missing) serialize to an ISO string or `null`. A single malformed or partial document can no longer throw and 500 the endpoint.
- Ownership is unchanged and correct: every read is scoped to `users/{uid}/aiConversations` with the UID taken from the verified Firebase token. No hardcoded UID; the unauthenticated path still returns 401. There is **no** catch-and-return-empty anywhere in the path.

## 3. Morning alarm — root cause

The alarm stopped after roughly two rings because the re-ring was armed **once** and never re-armed. After the first dismissal a single retry alarm was scheduled, but when that retry fired nothing scheduled the next one, so the loop died. There was also no guard at the actual AlarmManager execution point to decide whether a pending alarm should still ring given the current session state, so behavior depended on stale, non-persistent assumptions.

## 4. Morning alarm — fix

Rebuilt as a persisted, date-keyed state machine (native Kotlin + a mirrored pure-Dart model):

- **Indefinite re-arm:** `AlarmReceiver.onReceive` re-arms the next 5-minute re-ring (`MorningSession.GAME_OPEN_DEADLINE_MILLIS`) from inside the receiver on every cycle, so `dismiss → 5-min deadline → game-not-opened → ring again` repeats with **no attempt cap** (no 2/3/5 limit).
- **Execution-point race guard:** before ringing, the receiver checks persistent state at the AlarmManager execution point — if the session is from a previous day it resets, if `COMPLETED` it does nothing, if `GAME_RUNNING` it does nothing, otherwise it rings and re-arms. This guarantees an old pending retry can never ring while the game is running or after completion.
- **Game start suppression:** opening the morning game (`MainActivity.gameStarted`) cancels the pending re-ring, stops sound/notification, and sets `GAME_RUNNING`.
- **Completion:** `gameCompleted` cancels all retries, stops notifications, clears temporary deadline/retry state, sets `COMPLETED`, and is idempotent against duplicate callbacks.
- **Fresh day:** sessions are keyed by `yyyy-MM-dd`; the primary fire stamps a fresh `sessionDate` and clears session fields so stale previous-day `COMPLETED`/`GAME_RUNNING` cannot leak into a new morning. The primary-fire path still reschedules repeating alarms (boot/`BootReceiver` behavior preserved).
- The re-ring reuses a single `(alarm, rering)` PendingIntent slot with `FLAG_UPDATE_CURRENT | FLAG_IMMUTABLE` so retries replace rather than accumulate, and `setAlarmClock` keeps it Doze-exempt.

## 5. Morning game — root cause

The game could complete in seconds because completion was gated only on finishing the challenge rounds, with no minimum-duration floor. A fast user (or an auto-advance) satisfied the completion condition well before any meaningful wake-up time, and nothing tied completion to elapsed wall-clock time.

## 6. Morning game — fix

- **Hard 10-minute wall-clock floor:** `GameSession` computes `elapsed = now - startedAt` and gates completion on `canComplete = allChallengesFinished && elapsed >= 10 minutes`. Completion is impossible at 30s / 1m / 3m / 5m / 9m59s regardless of how fast the challenges are finished.
- **Survives lifecycle:** `startedAt` is persisted in SharedPreferences keyed by `alarmId + yyyy-MM-dd`, so backgrounding, activity recreation, or process restart recompute elapsed from real time and never reset the clock or auto-complete.
- **Keeps the session alive early:** when all planned stages finish before the floor, `_onStageCompleted` appends another non-math round (avoiding an immediate repeat) and keeps the session active; a 1-second ticker only refreshes the UI and re-checks the gate. `_completeSession` re-appends a round if invoked before the floor.
- **Non-math only:** `GameRegistry.pickRandom` draws exclusively from the cognitive/reaction/memory `morningPool` and filters any requested ids down to it, so the arithmetic games (`numberSequence`, `logicPuzzle`) can never be selected for the morning session. Those games remain registered for non-morning use (nothing was deleted or duplicated).
- Completion fires `reportGameCompleted` only once `canComplete` is true, and the persisted `startedAt` is cleared on genuine completion so the next morning starts fresh.

## 7. Files changed

Backend (`main`, commit `30f5380`):
- `src/repositories/aiConversationRepository.ts` — dropped composite-index title query; added pure null-safe mappers/title helper.
- `src/__tests__/aiHistory.test.ts` (new) — history mapping + auth-gate regression tests.

Frontend (`master`, commit `e34bf75`):
- `android/app/src/main/kotlin/com/fittrack/fittrack/alarm/AlarmModels.kt` — session states, `MorningSession` helpers + 5-min constant, session runtime fields; stale comment corrected.
- `android/app/src/main/kotlin/com/fittrack/fittrack/alarm/AlarmActivity.kt` — dismiss → 5-min deadline wiring; stale "2-minute re-ring" comment corrected.
- `android/app/src/main/kotlin/com/fittrack/fittrack/alarm/AlarmReceiver.kt` — execution-point race guard + indefinite re-arm.
- `android/app/src/main/kotlin/com/fittrack/fittrack/alarm/AlarmScheduler.kt` — 5-minute re-ring via `setAlarmClock`, reused PendingIntent slot.
- `android/app/src/main/kotlin/com/fittrack/fittrack/alarm/AlarmStore.kt` — persisted `mutate` helper + session fields.
- `android/app/src/main/kotlin/com/fittrack/fittrack/MainActivity.kt` — `gameStarted` / `gameCompleted` channel handlers.
- `lib/features/alarm/alarm_session_state.dart` — pure Dart state model.
- `lib/features/games/game_session.dart` — 10-minute wall-clock gate.
- `lib/features/games/game_registry.dart` — non-math morning allow-list in `pickRandom`.
- `lib/features/games/presentation/morning_challenge_screen.dart` — persisted `startedAt`, append-round loop, floor-gated completion.
- `test/ai_history_test.dart`, `test/alarm_session_state_test.dart`, `test/game_session_test.dart`, `test/game_registry_test.dart` — regression tests.

No forbidden files were changed or committed: `google-services.json`, `lib/firebase_options.dart` project values, `applicationId`, Android signing config, and dependency manifests (`pubspec.yaml`, `pubspec.lock`, `package.json`, `package-lock.json`) are untouched (they remain untracked/unmodified and were deliberately not staged). The `node_modules/.vite/vitest/results.json` cache artifact was left unstaged.

## 8. Tests added/updated

Backend:
- `src/__tests__/aiHistory.test.ts` (new, 8 tests): user-with-history, empty history, malformed/partial doc, invalid/missing timestamps, title-derivation rule, defensive message mapping, success-envelope shape, and the 401 auth gate (real express + fetch server).

Frontend:
- `test/ai_history_test.dart` (new): repo hits `/api/ai/conversations` and `/api/ai/conversations/:id/messages`, parses populated and empty lists, tolerates null `updatedAt`, empty title (falls back), missing `messageCount`, and garbage field types without crashing.
- `test/alarm_session_state_test.dart` (new): first fire, 5-minute deadline on dismiss, refire when game not opened, no-cap retry loop (continues well beyond 2/3/5), game-open cancels retry, no-ring-while-running, completion cancels all retries, idempotent completion, next-day reset.
- `test/game_session_test.dart` (new): no completion at 30s/1m/3m/5m/9m59s, finishing all challenges early does not bypass the minimum, completion at exactly 10m and later, unfinished challenges block completion past 10m, elapsed is a real wall-clock delta from a persisted `startedAt`.
- `test/game_registry_test.dart` (updated): never returns a math game even when a math game is explicitly preferred (100 draws), morning pool excludes `numberSequence`/`logicPuzzle`, fallback and avoid-repeat behavior.

## 9. flutter analyze result

`flutter analyze` — **clean**: "No issues found!" (ran in ~13s).

## 10. flutter test result

`flutter test` — **all passed**: **90 tests passed, 0 failed** (baseline 70 + the new alarm/game/history tests). Includes `ai_history_test.dart`, `alarm_session_state_test.dart`, `game_session_test.dart`, and the updated `game_registry_test.dart`.

## 11. backend typecheck result

`npm run typecheck` (`tsc -p tsconfig.json --noEmit`) — **clean**, exit code 0, no errors.

## 12. backend test result

`npm test` (`vitest run`) — **all passed**: **94 tests across 16 files, 0 failed** (baseline 86 + 8 new in `aiHistory.test.ts`).

## 13. Remaining risks

- **Native Android alarm behavior is NOT verified here.** The real device-level behavior of `AlarmManager`/`AlarmReceiver` — the indefinite re-ring chaining across cycles, Doze/standby exemption via `setAlarmClock`, exact-alarm permission behavior, and re-scheduling after a device reboot (`BootReceiver`) — cannot be exercised under `flutter test`. Only the mirrored pure-Dart state model (`alarm_session_state.dart`) and the Kotlin logic (by inspection) are verified; on-device confirmation requires a physical device/emulator.
- **The real 10-minute game run is NOT wall-clock verified end-to-end on a device.** The tests assert the completion gate against synthetic/persisted `startedAt` values and confirm the floor logic; they do not run an actual 10-minute session with real UI, backgrounding, and process death on hardware. Device testing is needed to confirm the lived experience (notification suppression while playing, re-ring suppression, completion marking the morning cycle permanently done).
- Only the Dart and backend logic plus their automated tests are verified in this pass. The native receiver retry chaining, Doze behavior, and boot rescheduling, and the real 10-minute game session, remain device-only and unverified.
- Minor: Firestore CRLF/LF normalization warnings appear on commit (cosmetic, Windows line-ending autocrlf); the optional defense-in-depth composite index was intentionally not added since the fixed query no longer needs any composite index.
