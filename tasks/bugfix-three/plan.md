# Implementation Plan — FitTrack three-bug fix

Grounded in `bugfix-audit.md`. Ordered by dependency; each item leaves the repo buildable. Verification uses the project's real commands:
- Backend: `npm run build` and `npm test` in `backend/`.
- Frontend: `flutter analyze` and `flutter test` in `frontend/`.
- Native Kotlin alarm logic: `flutter build apk --debug` must still COMPILE (do NOT ship an APK); runtime behavior is manually verified (steps in item 7).

## Hard constraints (every item must honor)
- Never modify `google-services.json`, `firebase_options.dart` project values, `applicationId`, signing config, or Firebase project config. Editing `firebase/firestore.indexes.json` IS allowed.
- No dependency upgrades or new packages (`pubspec.yaml`, `package.json` unchanged).
- Never remove auth checks, hardcode UIDs, fake success, or hide backend exceptions behind empty-state UI.
- Do not touch unrelated features (workout, nutrition, routine, water, dashboard, profile, progress, auth/FCM).
- Reuse existing services/providers/architecture; no duplicate alarm systems or game engines.

---

### BUG 1 — AI chat history 500

- [ ] 1. Make the conversation-title query in `listConversations` index-free (the proven root cause). Remove the composite `where('role','==','user').orderBy('createdAt','asc')` and instead read the first message with `orderBy('createdAt','asc').limit(1)` only (covered by the existing `messages (createdAt ASC)` index); derive the title from that doc, falling back to the generic label if it is missing or not a user turn. Keep owner-scoping and `toIso` null-handling unchanged. Do NOT wrap the endpoint in a catch-and-return-empty.
      Files: `backend/src/repositories/aiConversationRepository.ts`
      Verify: `npm run build` in `backend/` succeeds; the new tests in item 3 pass with `npm test`.

- [ ] 2. Add defense-in-depth composite index `messages (role ASC, createdAt ASC)` with `COLLECTION` queryScope to the indexes file so any future role-filtered+ordered query is covered. (indexes.json only — not Firebase project config.)
      Files: `firebase/firestore.indexes.json`
      Verify: file is valid JSON (`node -e "require('./firebase/firestore.indexes.json')"` from repo root exits 0); `npm run build` in `backend/` unaffected.

- [ ] 3. Add backend regression tests for the history read path. Extract the conversation→summary and message→DTO mapping into a pure, exported helper in the repository (or test via a `vi.mock` of Firestore like `aiConversationService.test.ts`) so no real Firestore is needed. Cover: (a) user with history maps correctly, (b) empty history → `[]`, (c) malformed/partial doc → safe fallbacks no throw, (d) missing/invalid `createdAt`/`updatedAt` → `toIso` null no throw, (e) success envelope shape, (f) auth-failure path returns 401 (extend the `authenticate` middleware pattern from `middleware.test.ts`).
      Files: `backend/src/__tests__/aiHistory.test.ts` (new); possibly a small exported mapper in `backend/src/repositories/aiConversationRepository.ts`.
      Verify: `npm test` in `backend/` — all new tests pass; existing suites stay green.

- [ ] 4. Add/extend frontend history-parsing tests: response with null `updatedAt`, empty `title`, missing `messageCount`, empty list, and populated list all parse without crashing; confirm the repo hits `/api/ai/conversations` and `/api/ai/conversations/:id/messages`.
      Files: `frontend/test/ai_history_test.dart` (extend)
      Verify: `flutter test test/ai_history_test.dart` in `frontend/` passes; `flutter analyze` clean.

---

### BUG 3 — Game session model + hard 10-minute minimum + non-math (do before alarm wiring so the alarm can consume the game signals)

- [ ] 5. Add a non-math morning game pool and a testable `GameSession` model. (a) Define a morning allow-list excluding `numberSequence` and `logicPuzzle`; route `ChallengeSession.build` / the morning pick through it so a math game can never be selected for the morning challenge (keep `GameRegistry` general-purpose, but the morning path uses the allow-list). (b) Add an immutable `GameSession` (`startedAt`, `currentRound`, `score`, `elapsedTime` getter from `DateTime.now().difference(startedAt)`, `minimumDuration` = `Duration(minutes:10)`, `isRunning`, `isCompleted`, and `canComplete(allChallengesFinished)` = `allChallengesFinished && elapsed >= minimumDuration`). Elapsed is always a wall-clock delta from a persisted `startedAt`, never a countdown.
      Files: `frontend/lib/features/games/challenge_session.dart` (morning pool/allow-list), `frontend/lib/features/games/game_session.dart` (new model). Reuse `GameDifficulty`/`GameId` from existing files.
      Verify: `flutter analyze` clean; unit tests in item 6 pass.

- [ ] 6. Enforce the 10-minute gate in the morning challenge screen using `GameSession`. When all stages finish before 10 minutes, keep the session active and generate additional non-math rounds (reusing the allow-list pool) instead of completing; only call `reportGameCompleted` when `canComplete` is true. Persist `startedAt` (e.g. SharedPreferences keyed by session date) so backgrounding/activity recreation recomputes elapsed from real time and never auto-completes on pause. Report `reportGameStarted` on first genuine interaction. Keep the existing anti-bypass `PopScope(canPop: _completed)`.
      Files: `frontend/lib/features/games/presentation/morning_challenge_screen.dart`; small helper use of `shared_preferences` (already a dependency).
      Verify: `flutter test test/game_session_test.dart` passes (added in this item or item 5): cannot complete < 10 min; finishing stages early keeps session active and adds rounds; elapsed computed from `startedAt`; completes after 10 min when final condition met; state survives recomputation from persisted `startedAt`. Also extend `frontend/test/game_registry_test.dart` / `challenge_session_test.dart` to assert the morning pool excludes `numberSequence` and `logicPuzzle`. `flutter analyze` clean.

---

### BUG 2 — Alarm state machine (depends on item 6's gameRunning/gameCompleted signals)

- [ ] 7. Reimplement the alarm re-ring as an indefinite, state-guarded loop in native Kotlin with a 5-minute game-open deadline, keyed by morning-session date. (a) In `AlarmReceiver.onReceive`, before ringing, read persisted session state and apply the race rule: `if morningRoutineCompleted → return; else if gameRunning → return; else ring AND re-arm the next 5-min re-ring`. (b) Change the deadline to 5 minutes for the challenge cycle (do not depend on the 20-min `challengeTimeoutSeconds` default; use a 5-min retry constant for the re-ring cycle). (c) Re-arm from inside the receiver each cycle so retries continue with NO cap. (d) Use `setExactAndAllowWhileIdle` (or keep `setAlarmClock`) so Doze doesn't drop the retry; keep PendingIntent request codes unique per (alarm, rering) and reuse/replace (no duplicate accumulation); make the receiver idempotent. (e) Persist session id/date, state, game-open deadline, game-running flag, game start time, completion flag in `AlarmStore`; reset to a fresh session when the date rolls over. (f) On `gameStarted` set gameRunning + cancel pending retry; on `gameCompleted` set completed + cancel all retries + stop notifications + clear deadline/retry state + guard against duplicate completion. Preserve `BootReceiver` reschedule behavior.
      Files: `frontend/android/app/src/main/kotlin/com/fittrack/fittrack/alarm/AlarmReceiver.kt`, `AlarmScheduler.kt`, `AlarmStore.kt`, `AlarmModels.kt`, `AlarmActivity.kt`, `MainActivity.kt`.
      Verify: `flutter build apk --debug` COMPILES (do not install/ship). Manual on-device verification (document results): alarm fires; dismiss → rings again after 5 min if game not opened; dismiss again → another 5-min cycle; continues on 3rd/4th/5th with no cap; opening the game cancels the pending retry and no ring occurs while the game runs; completing the game cancels all retries and no further ring that morning; next day starts a fresh session; reboot reschedules the alarm.

- [ ] 8. Add a pure Dart alarm **state model** that mirrors the native transitions so the logic is unit-testable under `flutter test` (native Kotlin can't run here). Model: `idle → alarmRinging → alarmDismissed → waitingForGame → gameRunning → completed`, 5-min deadline, indefinite retry, session keyed by date.
      Files: `frontend/lib/features/alarm/alarm_session_state.dart` (new, pure Dart), `frontend/test/alarm_session_state_test.dart` (new).
      Verify: `flutter test test/alarm_session_state_test.dart` passes: first fire; dismiss starts 5-min deadline; no game after 5 min → fires again; retry continues on 3rd/4th/5th with no cap; game opens → retry cancelled & no ring while running; game completes → all retries cancelled; completed morning → no duplicate completion; next-day session resets. `flutter analyze` clean.

---

### Final verification

- [ ] 9. Run the full gates and confirm no unrelated feature regressed.
      Files: none (verification only).
      Verify: `backend/` → `npm run build` and `npm test` both green. `frontend/` → `flutter analyze` clean and `flutter test` all green. `frontend/` → `flutter build apk --debug` compiles (do NOT produce/ship a release APK). Confirm no edits to `google-services.json`, `firebase_options.dart`, `applicationId`, signing config, `pubspec.yaml`, or `backend/package.json` (`git diff --stat` in each repo).

## Test matrix summary
- Backend (vitest): history for user-with-data, empty, malformed doc, invalid/missing timestamp, auth failure 401, success envelope shape (items 3).
- Frontend history (flutter_test): provider/repo parsing with null/missing fields, empty + populated lists (item 4).
- Game (flutter_test): no completion < 10 min; early finish does not bypass minimum; elapsed from `startedAt`; completion after 10 min; lifecycle-safe; morning pool is non-math (items 5–6).
- Alarm Dart state model (flutter_test): first fire; 5-min deadline on dismiss; re-fire when game not opened; indefinite retry (3rd/4th/5th, no cap); game-open cancels retry; no ring while gameRunning; completion cancels all; no duplicate; next-day reset (item 8). Native Kotlin receiver re-arm + execution-point guard: manual on-device steps (item 7).
