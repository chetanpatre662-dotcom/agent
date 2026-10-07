# Implementation Plan — Blank Tabs Bug Fix

## Root Cause

**File:** `lib/features/shell/home_shell.dart`, lines 213–215 (class `_NavTab`)

The `_NavTab` widget uses:
```dart
ConstrainedBox(
  constraints: const BoxConstraints(minHeight: AppSpacing.minTapTarget),  // min 48, max ∞
  child: Center(  // Center is greedy — expands to fill max available height
    child: AnimatedContainer(...)
  ),
)
```

`Scaffold` lays out `bottomNavigationBar` **before** the body and gives it **loose** height constraints (`0 ≤ h ≤ screenHeight`). The `ConstrainedBox` with only a `minHeight` (no `maxHeight`) passes `48 ≤ h ≤ screenHeight` to `Center`. `Center` is a greedy widget — it sizes itself to the **maximum** available height. All 5 `Expanded` tabs in the `Row` do this, so the `Row` becomes `screenHeight – padding` tall, the bottom nav consumes the **entire screen**, and `Scaffold` allocates **zero height** to the body's `IndexedStack`. Every tab screen gets `800 × 0` constraints and renders nothing.

**Reproduced in widget test:**
- BUG: Nav = `Size(800.0, 600.0)`, Body = `Size(800.0, 0.0)` — matches the blank-content symptom exactly
- FIX: Nav = `Size(800.0, 64.0)`, Body = `Size(800.0, 536.0)` — content is visible

**Introduced by:** commit `fe21590` ("feat: redesign bottom nav as Midnight Energy premium bar"), which replaced the built-in `NavigationBar` (which enforces its own height) with a custom `_MidnightNavBar` using the unconstrained `ConstrainedBox` + `Center` pattern.

**This affects ALL 5 tabs** (Home, Workout, Nutrition, AI Coach, Profile) because they all share the same shell `Scaffold.body` / `IndexedStack` that receives zero height.

---

## Hard Constraints (carry forward — do NOT violate)

- Do NOT redesign anything or restart the Midnight Energy work
- Do NOT replace real screens with mock/placeholder widgets
- Do NOT create duplicate screens
- Do NOT remove existing functionality
- Keep the Midnight Energy theme exactly as-is (colors, gradients, typography, visual identity)
- Do NOT modify alarm or native logic
- Preserve all existing providers, APIs, Firebase, authentication, navigation logic, and all feature functionality

---

## Plan

- [ ] 1. **Fix the `_NavTab` `ConstrainedBox` to cap its max height, preventing `Center` from expanding to fill the screen.**

      In `_NavTab.build()` (line 214), change:
      ```dart
      constraints: const BoxConstraints(minHeight: AppSpacing.minTapTarget),
      ```
      to:
      ```dart
      constraints: const BoxConstraints(
        minHeight: AppSpacing.minTapTarget,
        maxHeight: AppSpacing.minTapTarget,
      ),
      ```
      This is a 1-line change (adding `maxHeight`). It makes each tab cell a fixed 48px height, which matches the Material accessibility minimum and is the same height the built-in `NavigationBar` used. The `Center` widget still centers the pill/icon within the 48px cell, but can no longer expand beyond it. The bottom nav bar will now be ~64px (48 + 16px padding + bottom safe area inset) instead of consuming the entire screen.

      **Files:** `lib/features/shell/home_shell.dart` (single line change, line 214)

      **Tabs affected:** All 5 (Home, Workout, Nutrition, AI Coach, Profile) — the fix is in the shared nav bar that was starving ALL tabs of height.

      **Verify:** `flutter analyze` — no new issues. `flutter test` — all 91 existing tests still pass.

- [ ] 2. **Add a regression widget test proving the bottom nav bar does NOT consume all available height and the IndexedStack body has non-zero height.**

      Create `test/home_shell_nav_height_test.dart` with two tests:
      - Test 1: Reproduce the original bug pattern — a `Scaffold` with a `bottomNavigationBar` using `ConstrainedBox(minHeight: 48)` + `Center` (no maxHeight) → assert body height is 0 (documents the bug).
      - Test 2: The fixed pattern — `ConstrainedBox(minHeight: 48, maxHeight: 48)` + `Center` → assert body height > 0 and nav height < 100px.
      - Test 3: Pump the actual `_MidnightNavBar` widget structure (or the `HomeShell` with mocked providers if feasible) and assert the `IndexedStack` body has non-zero height.

      **Files:** `test/home_shell_nav_height_test.dart` (new file)

      **Verify:** `flutter test test/home_shell_nav_height_test.dart` — all new tests pass, confirming the regression cannot recur.

- [ ] 3. **Per-tab content verification walkthrough.**

      With the fix applied, verify each tab screen renders real content (not blank):

      - **Home (tab 0):** `DashboardScreen` builds a `Scaffold` → `SafeArea` → `RefreshIndicator` → `ListView` with the greeting, streak banner, AI insight, summary card, routine card, quick actions, water widget, reminders widget. With non-zero body height, `ListView` has real constraints and renders visible content.
      - **Workout (tab 1):** `WorkoutHubScreen` builds a `Scaffold` → `SafeArea` → `RefreshIndicator` → `ListView` with the "Start empty workout" button, templates section, and recent workouts. Same sizing resolution.
      - **Nutrition (tab 2):** `NutritionScreen` builds `Scaffold` → `SafeArea` → `RefreshIndicator` → `ListView` with macro summary, meal sections, add-meal button.
      - **AI Coach (tab 3):** `AiCoachScreen` builds `Scaffold` → `AppBar` with `TabBar` → `Container(decoration: gradient)` → `TabBarView` with `AiChatView` and `AiRecommendationsView`. The `TabBarView` needs bounded height from its parent, which it now gets.
      - **Profile (tab 4):** `ProfileScreen` builds `Scaffold` → `SafeArea` → `RefreshIndicator` → `ListView` with identity hero, stats, settings groups.

      All 5 screens use standard `Scaffold.body` → scrollable patterns that work correctly with any non-zero bounded height. The ONLY issue was the body receiving zero height from the nav bar stealing it all.

      **Verify:** `flutter analyze` — clean. `flutter test` — all tests pass (existing 91 + new regression tests).

---

## Summary

| Item | What | File(s) |
|------|------|---------|
| Fix  | Add `maxHeight: AppSpacing.minTapTarget` to `_NavTab`'s `ConstrainedBox` | `lib/features/shell/home_shell.dart` line 214 |
| Test | Regression test for nav bar height | `test/home_shell_nav_height_test.dart` (new) |

**Total change: 1 line modified + 1 new test file.** Smallest possible fix. No redesign, no functional changes, no theme changes.
