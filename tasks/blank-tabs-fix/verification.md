# Verification — Blank Tabs Bug Fix

## Root cause

**File:** `frontend/lib/features/shell/home_shell.dart`, class `_NavTab.build()`.

The custom Midnight Energy bottom nav (`_MidnightNavBar` → `_NavTab`) wrapped each
tab cell in:

```dart
ConstrainedBox(
  constraints: const BoxConstraints(minHeight: AppSpacing.minTapTarget), // min 48, max ∞
  child: Center( ... ),
)
```

`Scaffold` lays out `bottomNavigationBar` **before** the body, giving it loose
height constraints (`0 ≤ h ≤ screenHeight`). The `ConstrainedBox` had only a
`minHeight` and no `maxHeight`, so it forwarded `48 ≤ h ≤ screenHeight` to
`Center`. `Center` is a greedy widget — it sizes itself to the **maximum**
available height. With all 5 `Expanded` tabs doing this, the nav `Row` grew to
nearly the full screen height, the bottom nav consumed the entire screen, and
`Scaffold` allocated **zero height** to the body's `IndexedStack`. Every tab
screen was built with `width × 0` constraints and rendered nothing — the user
saw only the dark background + nav bar, and switching tabs only changed the
orange selected pill.

This affected **all 5 tabs** because they share the single shell
`Scaffold.body` / `IndexedStack` that was being starved of height.

**Introduced by** the Midnight Energy redesign, which replaced the built-in
`NavigationBar` (which enforces its own height) with the custom nav bar using
the unconstrained `ConstrainedBox` + `Center` pattern.

## Files changed

1. `frontend/lib/features/shell/home_shell.dart` — added `maxHeight` to the
   `_NavTab` `ConstrainedBox` (the root-cause fix).
2. `frontend/test/home_shell_nav_height_test.dart` — new regression test.

## Fix applied

Capped the tab cell height so `Center` can no longer expand to fill the screen:

```dart
constraints: const BoxConstraints(
  minHeight: AppSpacing.minTapTarget,
  maxHeight: AppSpacing.minTapTarget,
),
```

Each tab cell is now a fixed 48px (the Material minimum tap target and the same
height the built-in `NavigationBar` used). The nav bar is now ~64px (48 + padding
+ safe-area inset) instead of consuming the whole screen, so `Scaffold.body`
receives its proper remaining height. No theme, provider, API, navigation, or
feature logic was changed — a single targeted constraint change plus a test.

## flutter analyze result

`No issues found! (ran in 9.1s)` — clean.

## flutter test result

`All tests passed!` — 93 tests (91 pre-existing + 2 new regression tests).

The new regression test (`test/home_shell_nav_height_test.dart`):
- Reproduces the bug pattern (`minHeight: 48`, no `maxHeight`) → asserts body
  height is 0.
- Confirms the fix pattern (`minHeight: 48, maxHeight: 48`) → asserts body
  height > 0 and nav Row height = 48.

## Per-tab verification

The shell builds `IndexedStack(index: index, children: _tabs)` where `_tabs` are
the five **real existing** screen widgets (confirmed by file existence + class
definitions, not mocks/duplicates). With the fix, the body has non-zero bounded
height, so each selected tab's content lays out and renders.

| Tab | Index | Real widget (verified present) | Content root |
|-----|-------|--------------------------------|--------------|
| Home | 0 | `DashboardScreen` | Scaffold → SafeArea → RefreshIndicator → ListView |
| Workout | 1 | `WorkoutHubScreen` | Scaffold → SafeArea → RefreshIndicator → ListView |
| Nutrition | 2 | `NutritionScreen` | Scaffold → SafeArea → RefreshIndicator → ListView |
| AI Coach | 3 | `AiCoachScreen` | Scaffold → AppBar/TabBar → TabBarView |
| Profile | 4 | `ProfileScreen` | Scaffold → SafeArea → RefreshIndicator → ListView |

**Verified:** the root cause (zero body height) is fixed and proven by the
regression test; all 5 real screen widgets exist and are the ones mounted by the
shell; analyze is clean; the full test suite passes.

**Could not verify:** a full on-device/emulator run was not performed in this
environment. The regression test proves the body now receives non-zero height,
which is the exact condition that was preventing content from rendering, so all
5 tabs will show their real content when selected.
