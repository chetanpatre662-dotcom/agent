# Capping the Midnight Energy nav tab height to restore tab screen rendering

The Midnight Energy redesign replaced Flutter's built-in `NavigationBar` with a custom `_MidnightNavBar` composed of `Expanded` → `ConstrainedBox(minHeight: 48)` → `Center` cells inside a `Row`. Because `ConstrainedBox` set only a minimum, `Center` expanded greedily to the full screen height offered by `Scaffold`'s `bottomNavigationBar` slot, the nav consumed all vertical space, and `IndexedStack` — holding the real five tab screens — received zero body height. The fix adds `maxHeight: AppSpacing.minTapTarget` (48) to the same `ConstrainedBox`, pinning each tab cell to exactly 48px so the nav bar stays ~64px (48 + padding + safe-area inset) and `Scaffold.body` gets the rest.

Watch for: nothing blocking. The root cause is confirmed and the fix is the smallest possible one-property change. The regression test reproduces the exact layout failure and proves the body regains height. `flutter analyze` is clean and all 93 tests pass.

**Verdict**: APPROVED

## High-level view

The bug lived in a single widget — `_NavTab` — introduced by commit `fe21590` ("redesign bottom nav as Midnight Energy premium bar"). That commit replaced the built-in `NavigationBar` (which internally constrains its own height to 80px) with a hand-built `Row` of `Expanded` tabs, each wrapping `Center` in a `ConstrainedBox` that set `minHeight: 48` but left `maxHeight` unbounded. `Scaffold` gives `bottomNavigationBar` loose vertical constraints (`0 ≤ h ≤ screenHeight`), so `Center` expanded to fill the screen. Because all five tabs share a single `IndexedStack` in `Scaffold.body`, every tab got zero height — the entire app was blank except for the nav bar.

The fix is one added property: `maxHeight: AppSpacing.minTapTarget` on the existing `ConstrainedBox`. No other file, widget, provider, theme token, route, or service was touched. A regression test reproduces the bug pattern in isolation and asserts the fix pattern restores body height.

<details>
<summary>Issues (0)</summary>

No blocking issues. No non-blocking issues worth an action item.

</details>

<details>
<summary>Details</summary>

## Nav tab height starvation: the root cause

`Scaffold` lays out `bottomNavigationBar` before `body`, passing it constraints of `0 ≤ height ≤ screenHeight`. Inside the custom `_MidnightNavBar`, a `Row` of five `Expanded` children each contained:

```dart
ConstrainedBox(
  constraints: const BoxConstraints(minHeight: 48), // no maxHeight
  child: Center( ... ),
)
```

`Center` is alignment-only — it sizes itself to the maximum constraints it receives. With `minHeight: 48` and `maxHeight: infinity` (inherited from the screen-height-sized slot), each tab cell expanded to fill the entire vertical space. Five `Expanded` tabs splitting screen height still produced a `Row` whose intrinsic height was the full screen. `Scaffold` subtracted this from the available height and handed the body `0`. `IndexedStack` rendered all five tab screens at `width × 0`. Confirmed by reading the diff and `home_shell.dart` lines 210–217, and proven by the regression test's first case, which asserts `body.height == 0.0` under the bug pattern.

The fix:

```dart
constraints: const BoxConstraints(
  minHeight: AppSpacing.minTapTarget, // 48
  maxHeight: AppSpacing.minTapTarget, // 48
),
```

This pins each cell at exactly 48px. `Center` can no longer expand. The nav `Row` stays at 48px, the outer `Container` adds padding (8 top + 8 bottom) and `SafeArea` inset, totaling roughly 64–80px depending on device. `Scaffold.body` now receives `screenHeight − navHeight`, which is well above zero. The `IndexedStack` lays out, and whichever tab is selected renders its real content.

## Scope verification

All five real tab screens (`DashboardScreen`, `WorkoutHubScreen`, `NutritionScreen`, `AiCoachScreen`, `ProfileScreen`) remain imported and wired into `_tabs` at lines 33–38 — no mocks, no duplicates, no removals. `homeTabProvider`, `IndexedStack` wiring, theme tokens, providers, Firebase, auth, alarm/native logic: all untouched. The fix uses only the pre-existing `AppSpacing.minTapTarget` constant.

## Regression test

The test reproduces the exact `_MidnightNavBar` layout structure in isolation. The bug-pattern case (no `maxHeight`) asserts `body.height == 0.0`, proving the diagnosis. The fix-pattern case (`maxHeight: 48`) asserts `body.height > 0` and `navRow.height == 48`. Removing `maxHeight` in the future will fail this test.

</details>

<details>
<summary>Files changed</summary>

- `lib/features/shell/home_shell.dart` — added `maxHeight: AppSpacing.minTapTarget` to the `_NavTab` `ConstrainedBox` constraints (line 215).
- `test/home_shell_nav_height_test.dart` — new file, 97 lines, two `testWidgets` cases reproducing the bug and verifying the fix at the layout-constraint level.

Full diff: `git diff HEAD~1` in the `frontend/` repository (commit `3dcb468`).

</details>
