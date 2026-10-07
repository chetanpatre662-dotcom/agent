# Midnight Energy workout redesign + weight-input focus fix

Redesigns the four Workout screens (`workout_hub_screen`, `active_workout_screen`, `set_row`, `workout_timer`) onto the Midnight Energy design system and fixes the keyboard-focus bug where typing a digit into the Kg/Reps field dropped focus after the first character. The focus fix is the substantive logic change; everything else is a visual-layer swap that preserves controller, provider, repository, and navigation wiring. The diff is commit `3a5e159` reviewed against the working tree (the design-system foundation files it depends on — `app_shadows.dart`, `premium_widgets.dart`, `state_views.dart` — are present on disk and their signatures match every call site).

Watch for: the focus fix rests on a three-part invariant (stable `SetRow` key + always-single-child `_PreviousHint` + once-created controller/focusNode) — all three are present and the new widget test exercises the real rebuild-on-every-keystroke path (confirmed). No Firebase/provider/controller/repository code was touched; the multi-exercise picker loop is byte-for-byte intact (confirmed). Verification evidence is recorded in the commit message: analyze clean on the workout feature, the new focus test and existing workout tests pass (confirmed present, not re-run per instruction).

**Verdict**: APPROVED

## High-level view

The focus bug was an element-reconciliation problem, not a controller-text problem. Every keystroke routes through `updateSet`, emits new state, and rebuilds the whole `ActiveWorkoutScreen` subtree. `SetRow` was previously unkeyed and the async "Previous:" hint above it could add or remove a Column child the moment `previousPerformanceProvider` resolved, reparenting the `_NumField` State (and its `FocusNode`) mid-edit. The fix stabilizes identity on three fronts — a stable `ValueKey('setrow_<ex>_<set>')` on each row, a dedicated `_PreviousHint` that renders exactly one child in every AsyncValue state (zero-height box when empty), and the already-stable `late final` controller/focusNode guarded by `!hasFocus`. All three are necessary; all three are present. A widget test reproduces the parent-rebuilds-on-every-change condition and asserts focus retention plus the full "100" value.

Real workout data and behavior are preserved. The session summary computes exercise count, completed sets, and volume live from `workout.exercises` (falling back to stored aggregates when live volume is zero) and reuses the real `WorkoutElapsed` timer for duration — no mock data. `_addExercise`, `_finish`, `_cancel`, `PopScope` save/discard, `_openSession`'s load→push→invalidate contract, and the template `duplicate` + `dup.when` flow are untouched outside styling.

The multi-exercise selection path is not regressed: `_addExercise` still awaits `push<List<Exercise>>` into `ExerciseLibraryScreen(isPicking: true)` and loops `for (final ex in picked) _c.addExercise(ex)`. The redesign only restyles the button that triggers it.

Design-system usage is clean. All colors come from `AppColors` tokens (energyOrange, fieryRed, success, amber, surfaceHigh, divider, coolGray, mutedGray, white), spacing/radii from `AppSpacing`, depth from `AppShadows.card`/`glow`, and the reusable `GlowCard`/`StatNumber`/`SectionHeader`/`GradientButton`/`EmptyView`/`SkeletonBox` widgets. No invented constants, no leftover Material-blue/`primaryContainer`/`errorContainer`/`Card`/light surfaces in the touched files.

Keyboard and overflow safety are handled: `resizeToAvoidBottomInset: true` on the active screen, `SafeArea` on the hub, scrollable `ReorderableListView`/`ListView` bodies, `Expanded` number fields, and a `FittedBox` wrapper on summary metrics so large volume values scale down rather than overflow on small phones.

<details>
<summary>Issues (2)</summary>

1. **`_PreviousHint` empty-state box is non-zero width, not non-zero height** — in the empty/loading state it returns `SizedBox(height: 0, width: double.infinity)`, which is still exactly one Column child (so the fix holds), but naming it "zero-height box" is accurate while the `width: double.infinity` is load-bearing only for layout consistency. Non-blocking; no action required, noted for clarity.
2. **`BuildContext` used across `await` in template duplicate `onTap`** — pre-existing pattern carried through the restyle (`_openSession`/`ScaffoldMessenger.of(context)` after `await ...duplicate(...)`), unchanged from the original. Out of scope for this review (not newly introduced); flagged only so a future pass can add a `mounted` guard if analyze ever starts enforcing `use_build_context_synchronously` here.

</details>

<details>
<summary>Details</summary>

### Focus fix: identity stability across the rebuild-on-every-keystroke

The mechanism is confirmed from the diff. `onChanged` → `_c.updateSet` → new StateNotifier state → `ActiveWorkoutScreen.build` → `ReorderableListView.builder` → `_ExerciseCard.build` → `exercise.sets.asMap().entries.map(... SetRow ...)`. Before the fix, two things made the `_NumField` subtree fragile:

- `SetRow` was built without a `key`, so the parent matched rows positionally.
- The "Previous:" hint was inlined as `previous.maybeWhen(data: p==null ? SizedBox.shrink() : Padding(...), orElse: SizedBox.shrink())`. When `previousPerformanceProvider` resolved from loading to data (or null to non-null) while the user typed, the widget occupying that slot changed, shifting sibling structure in the Column.

The fix (confirmed in the diff):

```dart
...exercise.sets.asMap().entries.map(
      (e) => SetRow(
        key: ValueKey('setrow_${index}_${e.key}'),   // stable per set
        ...
      ),
    ),
```

```dart
// _PreviousHint — exactly one child in every AsyncValue state
if (text == null) return const SizedBox(height: 0, width: double.infinity);
return Padding(... Row(... Text(text) ...));
```

and `_NumFieldState` keeps `late final _controller`/`_focusNode` created once in `initState`, with `didUpdateWidget` syncing text only when `!_focusNode.hasFocus`. This is the necessary-and-sufficient combination: the controller guard alone (which already existed) does not prevent focus loss caused by State reparenting; the key plus the always-single-child hint remove the structural churn that caused the reparent. Reasoning directly from the diff, the stated root cause is correct and the fix matches it.

The `nf_Kg`/`nf_Reps`/`nf_Min`/`nf_Km` const keys on `_NumField` are reused across rows, but because each `SetRow` is now a distinct keyed element they only need to disambiguate within one row, where they are unique. No regression there.

### Focus test exercises the real condition

`test/workout_set_row_focus_test.dart` hosts a keyed `SetRow` whose `onChanged` mirrors `controller.updateSet` by calling `setState(() => _set = s)` — reproducing the parent-rebuilds-on-every-change path that dropped focus. It types `1`, `10`, `100` with pumps between, then asserts the `EditableText` holds `'100'`, `focusNode.hasFocus` is true, and the parsed value `100.0` flowed through `onChanged`. The test does not reproduce the async `_PreviousHint` transition specifically, but it does prove the rebuild-driven focus retention and the value-flow contract, which are the user-visible symptoms.

### Real-data session summary

`_WorkoutSummary` computes `completedSets` and `liveVolume` by folding over `workout.exercises` and falls back to `workout.totalVolume` only when live volume is zero (finished workouts recomputed server-side). Duration reuses the real `WorkoutElapsed` timer bound to `workout.startedAt`/`durationSeconds`/`running`. These are the real model fields from `models/workout.dart`; nothing is faked. Each metric is wrapped in a `FittedBox(scaleDown)` so a large volume value shrinks instead of overflowing a quarter-width column on a narrow phone.

### Timer logic preserved

`workout_timer.dart` appears as a new file in the diff (the repo's foundation files are largely untracked), but the reproduced logic matches the documented behavior: `formatDuration`, `WorkoutElapsed` count-up with a 1s periodic timer started/cancelled in `initState`/`didUpdateWidget`/`dispose`, and `RestTimerBar` countdown with Skip and +15s. Only the `RestTimerBar` visual layer changed — `primaryContainer` Material bar replaced with `surfaceHigh` + an energyOrange `LinearProgressIndicator` on a 15%-alpha track. `onDismiss`, `_remaining`, and the +15s handler are intact.

### Hub and set-row restyle

`workout_hub_screen` keeps `_openSession`'s load→push→invalidate flow, the template `duplicate(t.id, asTemplate:false)` + `dup.when(onSuccess/onFailure)` path, `_emptyWorkout()` fields, the `!w.isTemplate` history filter, and `DateFormat.MMMEd()`. `Card(ListTile)` became `GlowCard`, section titles became `SectionHeader`, and `_StatusChip` became a dark color-coded pill driven by a `_statusColor` switch (completed→success, inProgress→energyOrange, cancelled→fieryRed, paused→amber, planned→mutedGray). `set_row` swapped `errorContainer`/`colorScheme.primary`/`outline` for `fieryRed`/`success`/`mutedGray` and `energyOrange`, with number fields in `Expanded` to avoid horizontal overflow. The `Dismissible` key, cardio-vs-strength branching, and all `onChanged`/`onToggleComplete`/`onRemove` wiring are unchanged.

### Verification evidence (read, not re-run)

The commit message records: analyze clean on the workout feature and the new test ("No issues found"); the new focus test plus existing `workout_model`/`exercise_filter` tests pass; remaining full-project analyze/test failures are isolated to out-of-scope in-progress files (profile/nutrition/ai) from other steps. Per instruction I did not re-run analyze or the suite. The evidence is present and specific, so no narrow spot-check was warranted beyond confirming the referenced tokens/widgets exist on disk (they do).

</details>

<details>
<summary>File map</summary>

- `lib/features/workout/presentation/active_workout_screen.dart` — session summary header, GlowCard-style exercise cards, `_PreviousHint`, stable `SetRow` keys, orange CTAs, `EmptyView` empty state, `resizeToAvoidBottomInset`.
- `lib/features/workout/presentation/widgets/set_row.dart` — orange set-number badge, surfaceHigh number fields with orange focus border, success-green completion toggle, fieryRed swipe delete; controller/focus logic unchanged.
- `lib/features/workout/presentation/widgets/workout_timer.dart` — `RestTimerBar` restyled to surfaceHigh + orange progress; timer logic intact.
- `lib/features/workout/presentation/workout_hub_screen.dart` — GradientButton CTA, SectionHeaders, GlowCard template/history tiles, color-coded `_StatusChip`; `_openSession`/`duplicate` flows unchanged.
- `test/workout_set_row_focus_test.dart` — new widget test proving focus retention and full multi-digit value capture across parent rebuilds.

Full diff: `git show 3a5e159` in `frontend/`.

</details>
