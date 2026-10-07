# Implementation Plan — Workout feature "Midnight Energy" redesign + weight-input focus fix

## Scope and intent

Redesign the Workout screens to the existing Midnight Energy design system (orange/red fitness identity) and fix the weight/reps keyboard-focus bug in the set row. UI + the one focus-bug fix only. Do NOT touch Firebase, backend, providers, repositories, or business logic. Do NOT replace real data with mock data. Do NOT regress navigation or the multi-exercise picker.

## Baseline (recorded before any change)

- `flutter analyze` → **clean**: "No issues found!" (ran in ~10s). Any new analyzer issue is a regression introduced by this work.
- `flutter test` → **all pass** ("All tests passed!"). Existing suite includes `test/workout_model_test.dart` (model/volume/serialization) and `test/exercise_filter_test.dart`. The FCM test prints an expected "network down" line — that is not a failure.
- Flutter project root for all commands: `c:\Users\cheta\OneDrive\Desktop\fit\frontend`.
- Dart SDK uses modern features already in the codebase (e.g. `?trailing` null-aware element, `switch` expressions, `Color.withValues`). Match these; do not downgrade syntax.

## What each in-scope file currently does

### `lib/features/workout/presentation/workout_hub_screen.dart` (`WorkoutHubScreen`, ConsumerWidget)
- Workout hub. `AppBar(title: 'Workouts')` + `RefreshIndicator` wrapping a `ListView(padding: AppSpacing.lg)`.
- "Start empty workout" `FilledButton.icon` → `_openSession(context, ref, _emptyWorkout())`. `_emptyWorkout()` builds a `Workout(id:'', name:'New workout', type: strength, status: planned)`.
- `_openSession` calls `ref.read(activeWorkoutControllerProvider.notifier).load(workout)`, pushes `ActiveWorkoutScreen`, then on return invalidates `workoutsProvider` + `workoutTemplatesProvider`. **This load→push→invalidate flow is the navigation contract — keep it byte-for-byte.**
- Templates section: `templates.when(loading/error/data)`, each template a `Card(ListTile)`; tap duplicates via `workoutRepositoryProvider.duplicate(t.id, asTemplate:false)` then opens. Keep the repo call and `dup.when(onSuccess/onFailure)` logic.
- Recent workouts: `workouts.when(...)`, filters `!w.isTemplate`, empty → `EmptyView`, else list of `_WorkoutHistoryTile`.
- `_WorkoutHistoryTile`: `Card(ListTile)` with status `CircleAvatar`, name, subtitle built from `dateStr / completedSets / totalVolume / mins`, trailing `_StatusChip`. Uses `DateFormat.MMMEd()`.
- `_StatusChip`: Material `Chip` showing `status.label`.

### `lib/features/workout/presentation/active_workout_screen.dart` (`ActiveWorkoutScreen`, ConsumerStatefulWidget)
- Live session editor. Local state: `int? _restSeconds`. `_c` = `ref.read(activeWorkoutControllerProvider.notifier)`.
- `_addExercise()`: pushes `ExerciseLibraryScreen(isPicking:true)`, awaits `List<Exercise>?`, loops `_c.addExercise(ex)` for each. **This is the multi-exercise entry point — see "Multi-exercise selection path" below.**
- `_finish()`: confirm dialog → `_c.transition(WorkoutStatus.completed)`; on success reads `newPersonalRecords`, fires `reminderSchedulerProvider.notifyPersonalRecord(...)` per PR, invalidates `workoutsProvider`, pops `true`.
- `_cancel()`: confirm dialog → `_c.transition(WorkoutStatus.cancelled)`, pops `false`.
- `build`: watches `activeWorkoutControllerProvider`; `ref.listen` shows a SnackBar on new `failure`. If `workout == null` → `Scaffold(Center(CircularProgressIndicator))`. `isRunning = status == inProgress`.
- `PopScope(canPop:false, onPopInvokedWithResult:...)`: back shows a bottom sheet "Save & exit" / "Discard changes"; `save` → `_c.save()`, both → invalidate `workoutsProvider` + `navigator.pop()`. **Keep this exact save/discard back-handling.**
- Body: `AppBar(title: workout.name, actions:[WorkoutElapsed])`. `Column` → `Expanded` holding either `_EmptyExercises(onAdd:_addExercise)` or a `ReorderableListView.builder` of `_ExerciseCard` (key `ValueKey('ex_${ex.exerciseId}_$i')`, `onReorder:_c.reorderExercise`). Below: optional `RestTimerBar`, then a `SafeArea` row with "Add exercise" `OutlinedButton` + Finish (`GradientButton` with `AppColors.primaryGradient`) or Start (`FilledButton`). `persistentFooterButtons`: "Cancel workout" TextButton unless status is `planned`.
- `_ExerciseCard` (ConsumerWidget): `Card` → header row (name + `ReorderableDragStartListener` drag handle + `PopupMenuButton` remove), `previous.maybeWhen(data:...)` "Previous: {kg} × {reps}" hint from `previousPerformanceProvider(exercise.exerciseId)`, `Divider`, then `exercise.sets.asMap().entries.map(...)` → `SetRow` (CURRENTLY NO KEY on SetRow), then "Add set" TextButton. `_isCardio = primaryMuscle == cardio`.
- `_EmptyExercises`: centered emoji + icon + copy + "Add exercise" FilledButton.

### `lib/features/workout/presentation/widgets/set_row.dart`
- `SetRow` (StatelessWidget): a `Dismissible` (key `ValueKey('set_${exerciseIndex}_$setIndex')`, swipe-to-remove) wrapping a `Row`: set number, then two `_NumField`s. Cardio → Min/Km; strength → Kg/Reps. Each `_NumField` has a const key (`nf_Kg`, `nf_Reps`, `nf_Min`, `nf_Km`). Trailing completion `IconButton` (`check_circle`/`circle_outlined`). Values write back through `onChanged(set.copyWith(...))`.
- `_NumField` (StatefulWidget → `_NumFieldState`): already creates `_controller` and `_focusNode` once in `initState`; `didUpdateWidget` only syncs `_controller.text` from `widget.value` **when `!_focusNode.hasFocus`**; disposes both. `TextField` with `keyboardType: numberWithOptions(decimal:)`, centered, `onChanged: widget.onChanged`.

### `lib/features/workout/presentation/widgets/workout_timer.dart`
- `formatDuration(seconds)` helper. `WorkoutElapsed` (count-up timer from `startedAt`, uses `baseSeconds`, 1s `Timer`). `RestTimerBar` (countdown, `primaryContainer` Material bar, `LinearProgressIndicator`, Skip + "+15s"). Logic is correct — only the `RestTimerBar` visual layer needs restyling to the design system; keep both timers' timing logic intact.

## Supporting files (read; DO NOT change behavior)
- `active_workout_controller.dart`: `ActiveWorkoutController` (StateNotifier) with `load/addExercise/removeExercise/reorderExercise/addSet/removeSet/updateSet/toggleSetComplete/save/transition`. `addExercise` seeds one empty set. `updateSet` replaces one set and calls `_update`. **Every keystroke in a `_NumField` → `onChanged` → `onUpdateSet` → `_c.updateSet` → `state = copyWith(workout:...)` → the whole `ActiveWorkoutScreen` subtree rebuilds.** This rebuild-on-every-keystroke is the condition the focus fix must survive. Do not change the controller.
- `workout_providers.dart`: `workoutRepositoryProvider`, `workoutsProvider`, `workoutTemplatesProvider`, `workoutByIdProvider`.
- `previous_performance_provider.dart`: `previousPerformanceProvider` FutureProvider.family → last `ProgressionPoint?`. **Resolves asynchronously**, so the "Previous:" hint in `_ExerciseCard` can flip from empty to populated a frame or more after the card first builds — relevant to the focus bug (below).
- `models/workout.dart`: `ExerciseSet` (weightKg/reps/distanceM/durationSeconds/completed, `volume`, `copyWith` with `clearWeight`/`clearReps`), `WorkoutExercise` (`sets`, `volume` = sum of completed sets' volume), `Workout` (`completedSets`, `totalVolume`, `totalSets`, `totalReps`, `durationSeconds`, `status`, `exercises`). Use these REAL fields for the summary — do not invent.
- `models/exercise.dart`: `Exercise`, `MuscleGroup`, `ExerciseType`, `isCardio`.

## Multi-exercise selection path (MUST be preserved — trace)

1. `ActiveWorkoutScreen._addExercise` pushes `ExerciseLibraryScreen(isPicking: true)` and awaits `List<Exercise>?`.
2. `ExerciseLibraryScreen` (NOT in edit scope — do not modify): long-press enters `_multiSelectActive`, taps toggle `_selectedIds`; an extended FAB "Add N exercises" pops `Navigator.pop(picked)` with the `List<Exercise>`. Single tap (non-multi) pops `[ex]` (a one-item list).
3. Back in `_addExercise`, `for (final ex in picked) _c.addExercise(ex)` adds each.

**Preservation rule:** keep `_addExercise` returning/handling `List<Exercise>` and the `for` loop exactly. Do not change its push target, its `await ...push<List<Exercise>>`, or the loop. The redesign only restyles the button that triggers `_addExercise`; it must not change the picker contract.

## Weight-input focus bug — exact root cause

Symptom: typing a digit into the Kg/Reps field dismisses the keyboard / drops focus after the first character, so multi-digit values can't be entered.

Mechanism, precisely:
- Every keystroke calls `onChanged` → `_c.updateSet(...)` → StateNotifier emits new state → `ActiveWorkoutScreen.build` → `ReorderableListView.builder` → `_ExerciseCard.build` → the `exercise.sets.asMap().entries.map((e) => SetRow(...))` list is rebuilt.
- **`SetRow` is constructed WITHOUT a `key`.** The `Dismissible` *inside* it has a key, but the `SetRow` element itself is matched **positionally** by the parent `Column`.
- `_ExerciseCard` renders, ABOVE the sets, a conditional node from `previous.maybeWhen(data: p==null ? SizedBox.shrink() : Padding(...), orElse: SizedBox.shrink())`. `previousPerformanceProvider` is async; its state transitions (loading → data, data null → non-null) **change which widget occupies that slot and can change the children-list shape of the Column across rebuilds that happen while the user is typing**. Combined with un-keyed `SetRow`s, Flutter's element/State reconciliation for the `_NumField` subtree is fragile: the `_NumFieldState` (holding the `FocusNode`) can be matched to a different widget instance or rebuilt, which drops focus and dismisses the keyboard after the first keystroke.
- Secondary fragility: `nf_Kg`/`nf_Reps`/`nf_Min`/`nf_Km` keys are reused across every set in every exercise card. Keys only disambiguate within the same parent, so this is not the primary cause, but it removes the one signal that could stabilize matching when sibling structure shifts.

The existing `_NumField` stable-controller + `!hasFocus` guard is correct and necessary but **not sufficient**, because the focus loss comes from element reparenting/rebuild of the field's `State`, not from the controller text being overwritten.

## Fix strategy (robust, minimal, behavior-preserving)

Make the set-field subtree identity stable across the rebuild-on-every-keystroke so the `_NumFieldState` (and its `FocusNode`) is never reparented or recreated mid-edit, and ensure the async "Previous:" hint never shifts sibling positions.

Concrete requirements for the coder:
1. **Give each `SetRow` a stable `ValueKey`** keyed on identity that does not change while editing, e.g. `ValueKey('setrow_${exerciseIndex}_$setIndex')`, passed from `_ExerciseCard` (`key:` on the `SetRow(...)`). (The `Dismissible`'s own key stays as-is.)
2. **Keep `_NumField` controller/focus stable** (already done) AND keep the `didUpdateWidget` guard that only syncs when `!_focusNode.hasFocus`. Do not call `setState` on text change. Keep `onChanged: widget.onChanged`.
3. **Stabilize the "Previous:" hint slot** in `_ExerciseCard` so an async provider transition does not change the Column's child count/order: always render the hint position (e.g. a single widget that internally shows text or an empty/zero-height box via `previous.maybeWhen(...)` but is itself always present as one child, not spread/removed). Simplest: wrap the hint in a dedicated small widget that is always exactly one child in the Column regardless of loading/empty/data. This removes the structural shift while the user types.
4. Do **not** change `onChanged`'s payload (`set.copyWith(weightKg: double.tryParse(v), clearWeight: v.isEmpty)` etc.) — logged values must keep flowing to `_c.updateSet` unchanged.

This keeps controllers stable (req) and updates to provider state no longer replace controller text mid-edit (the guard) nor reparent the field (keys + stable hint slot).

## Design-system usage (tokens only — do NOT invent colors/constants)

- Workout identity: `AppColors.primaryGradient` (orange→red) for primary CTAs/heroes; `AppColors.energyOrange`, `AppColors.neonOrange`/`primary` (0xFFFF6B2C), `AppColors.fieryRed` for accents; `AppColors.success` (neon green) for completed sets.
- Surfaces: `AppColors.background`/`surface`/`surfaceHigh`/`divider`. Text: `AppColors.white`/`coolGray`/`mutedGray`.
- Reusable widgets (prefer over bespoke): `GlowCard`, `GradientCard`, `GradientButton` (defaults to primaryGradient), `StatNumber`, `SectionHeader`, `EmptyView`/`ErrorView`/`LoadingView`/`SkeletonBox`, `StreakBadge` (reference only).
- Spacing/radius via `AppSpacing` (lg=16 default padding; radiusLg/Xl/Xxl). Depth via `AppShadows.card`; glow via `AppShadows.glow(color)`.
- Mirror the Dashboard pattern: `Scaffold(body: SafeArea(... ListView padding AppSpacing.lg))`, `SectionHeader`, `provider.when(loading→skeleton / error→ErrorView / data)`, `StatNumber` for big numbers, `GlowCard`/`GradientCard` for cards, `GradientButton` for CTAs.

---

# Steps

- [ ] 1. **Fix the weight/reps focus bug in `set_row.dart` + `active_workout_screen.dart` and prove it with a widget test.**
      Add a stable `key: ValueKey('setrow_${exerciseIndex}_$setIndex')` to each `SetRow` created in `_ExerciseCard` (active_workout_screen.dart). In `set_row.dart` keep `_NumField`'s controller/focusNode created once in `initState`, keep the `didUpdateWidget` sync guarded by `!_focusNode.hasFocus`, and do not add `setState` on change. In `_ExerciseCard`, refactor the `previous.maybeWhen(...)` "Previous:" hint so it is ALWAYS exactly one child of the Column (a dedicated inline widget that renders text when data exists and a zero-height `SizedBox` otherwise) so async resolution of `previousPerformanceProvider` cannot change sibling structure while typing. Do not change any `onChanged` payloads or controller calls.
      Add a widget test `test/workout_set_row_focus_test.dart` that pumps a `SetRow` (or a minimal `_ExerciseCard`-like host that rebuilds the parent on each `onChanged`, mimicking `updateSet`), enters focus, types a multi-digit value character by character via `tester.enterText`/`sendKeyEvent` with pumps between, and asserts (a) the field still holds focus after each digit and (b) the final text is the full multi-digit string (e.g. "100"). Keep logged-value flow identical (assert `onChanged` received parseable values).
      Files: `lib/features/workout/presentation/widgets/set_row.dart`, `lib/features/workout/presentation/active_workout_screen.dart`, `test/workout_set_row_focus_test.dart`
      Verify: `cd c:\Users\cheta\OneDrive\Desktop\fit\frontend && flutter test test/workout_set_row_focus_test.dart` passes; then `flutter analyze` still clean.

- [ ] 2. **Redesign `set_row.dart` visual layer (keep the focus fix and all callbacks intact).**
      Restyle the row to the Midnight Energy system: set-number badge using `AppColors.surfaceHigh`/`energyOrange` text, `_NumField` inputs sitting on `AppColors.surface`/`surfaceHigh` with rounded `AppSpacing.radiusSm/Md` borders and orange focus accent (via the field's `InputDecoration` only — do not change `keyboardType`, `controller`, `focusNode`, or `onChanged`), completion toggle using `AppColors.success` when completed and `AppColors.mutedGray`/outline when not, and the Dismissible delete background using `AppColors.fieryRed`. Keep the `Dismissible` key, `onDismissed`, cardio vs strength branching, and all `onChanged`/`onToggleComplete`/`onRemove` wiring unchanged. Ensure no horizontal overflow on small phones (fields in `Expanded`, modest paddings).
      Files: `lib/features/workout/presentation/widgets/set_row.dart`
      Verify: `cd c:\Users\cheta\OneDrive\Desktop\fit\frontend && flutter test test/workout_set_row_focus_test.dart` still passes (focus fix intact) and `flutter analyze` clean.

- [ ] 3. **Restyle `RestTimerBar` in `workout_timer.dart` (visual only).**
      Replace the `primaryContainer` Material bar with a design-system bar: `AppColors.surfaceHigh` background with a subtle top divider/`AppShadows`, an orange timer icon (`AppColors.energyOrange`), white primary text + `coolGray` secondary, and a `LinearProgressIndicator` tinted `AppColors.energyOrange` on an `energyOrange.withValues(alpha:0.15)` track (ClipRRect rounded). Keep `WorkoutElapsed`'s `fontFeatures: tabularFigures` styling (optionally tint white). Do NOT change any timer logic, `Timer` setup, `_remaining`, `onDismiss`, `+15s`, `formatDuration`, or constructor signatures.
      Files: `lib/features/workout/presentation/widgets/workout_timer.dart`
      Verify: `cd c:\Users\cheta\OneDrive\Desktop\fit\frontend && flutter analyze` clean; `flutter test` all pass.

- [ ] 4. **Redesign `active_workout_screen.dart` — session summary header, exercise cards, CTA (preserve all flows).**
      Add `resizeToAvoidBottomInset: true` to the `Scaffold` (keyboard safety for the editable fields). Add a workout summary at the top of the exercises `Expanded` region (as the first item of the list or a header above it) built ONLY from real data: a row/grid of `StatNumber`s for Exercises (`workout.exercises.length`), Sets (`workout.completedSets` or count of completed sets across `exercises`), Volume (`workout.totalVolume` kg, or sum of `exercise.volume`), and Duration (via `WorkoutElapsed`/`formatDuration(workout.durationSeconds)`) — label each with `SectionHeader`-style accents. Convert `_ExerciseCard`'s `Card` to a `GlowCard` (surface + `AppShadows.card`), style the header name in white `titleMedium`, keep the drag handle (`ReorderableDragStartListener`) and remove `PopupMenuButton`, keep the ALWAYS-single-child "Previous:" hint from step 1 (styled with `AppColors.energyOrange`), replace the plain `Divider` with an `AppColors.divider` hairline, and keep the sets map (with the stable `SetRow` keys) and "Add set" button (orange text). Restyle the bottom action row: "Add exercise" as an outlined/`GlowCard` button that still calls `_addExercise` (multi-select preserved), Finish as the existing `GradientButton(gradient: AppColors.primaryGradient)`, Start restyled but still calling `_c.transition(WorkoutStatus.inProgress)`. Restyle `_EmptyExercises` using `EmptyView` (or a GlowCard variant) with an `energyOrange` accent and the same `onAdd` callback. Keep `PopScope` save/discard, `_finish`, `_cancel`, `ref.listen` SnackBar, `ReorderableListView` + `onReorder`, and all keys/providers unchanged. Ensure the list body scrolls and nothing clips when the keyboard is open.
      Files: `lib/features/workout/presentation/active_workout_screen.dart`
      Verify: `cd c:\Users\cheta\OneDrive\Desktop\fit\frontend && flutter analyze` clean; `flutter test` all pass (incl. the focus test).

- [ ] 5. **Redesign `workout_hub_screen.dart` — premium hub (preserve load→push→invalidate and repo calls).**
      Wrap body in `SafeArea`, keep `RefreshIndicator` + `ListView(padding: AppSpacing.lg)`. Replace the "Start empty workout" `FilledButton` with a prominent `GradientButton` (primaryGradient) that still calls `_openSession(context, ref, _emptyWorkout())` unchanged. Replace the emoji `Text` section titles with `SectionHeader` (Templates: `Icons.bookmark_outline`; Recent: `Icons.history`, with an `energyOrange`/`amber` accent). Convert template `Card(ListTile)` and `_WorkoutHistoryTile` `Card` into `GlowCard`s on `AppColors.surface`: history tile shows name (white), a stats row using real `completedSets`/`totalVolume`/duration (`StatNumber` or styled inline), status via a redesigned `_StatusChip` (dark pill, color-coded: completed→`success`, inProgress→`energyOrange`, cancelled→`fieryRed`, planned/paused→`mutedGray`/`amber`), and keep `onTap: _openSession(... w)`. Keep the template duplicate flow (`workoutRepositoryProvider.duplicate` + `dup.when`) and the `EmptyView`/`ErrorView` branches. Keep `DateFormat` usage. Do not change `_emptyWorkout()` fields or any provider wiring.
      Files: `lib/features/workout/presentation/workout_hub_screen.dart`
      Verify: `cd c:\Users\cheta\OneDrive\Desktop\fit\frontend && flutter analyze` clean; `flutter test` all pass.

- [ ] 6. **Final consistency + validation pass for the workout feature.**
      Re-read all four edited files for leftover old-theme usage within workout scope: raw `Card` without design intent, Material-blue defaults, `theme.colorScheme.primaryContainer` fills that should be `AppColors.surfaceHigh`, hard-coded colors/radii, or inconsistent spacing — bring them onto the design system WITHOUT changing behavior. Confirm no navigation/provider/controller call was altered (diff mentally against the "preserve" lists above), the multi-exercise `_addExercise` loop is intact, and keyboard/overflow safety holds (resizeToAvoidBottomInset, SafeArea, scrollable bodies, `Expanded` fields).
      Files: (review only; fix as needed) the four edited files.
      Verify: `cd c:\Users\cheta\OneDrive\Desktop\fit\frontend && flutter analyze` → "No issues found!"; `flutter test` → "All tests passed!". Both must match the clean baseline (plus the new passing focus test).

## Gaps / assumptions
- Summary metrics: for an in-progress session `workout.totalVolume`/`completedSets` may be 0 until recomputed server-side, so the summary should compute live values from `workout.exercises` (sum of `exercise.volume`, count of completed sets) and fall back to the `Workout` aggregate fields for finished workouts. Assumption recorded; coder picks the live computation for the active screen and the stored aggregates for the hub history tile.
- No widget test currently covers the active workout screen; step 1 adds a focused test for the bug only. Broader widget tests are out of scope to avoid touching provider wiring.
- `ExerciseLibraryScreen` is intentionally out of edit scope; the multi-select contract is preserved by leaving `_addExercise` untouched in behavior.
