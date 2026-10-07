# Implementation Plan — Bottom Navigation / App Shell Redesign (Midnight Energy)

Single-file, UI-only task. The ONLY file that changes is:

`frontend/lib/features/shell/home_shell.dart`

## Context verified during exploration

Token signatures were confirmed by reading the source (do not re-verify by assuming — these are exact):

- `app_colors.dart`: `AppColors.background` (0xFF0B0B12), `AppColors.surface` (0xFF1A1A27), `AppColors.surfaceHigh` (0xFF21212F), `AppColors.divider` (0xFF292936), `AppColors.white`, `AppColors.coolGray` (0xFFA7A7B5), `AppColors.mutedGray` (0xFF707080), `AppColors.primary` (0xFFFF6B2C), `AppColors.electricPurple` (0xFF7C3AED). Gradients: `AppColors.primaryGradient` (orange 0xFFFF8A00 → red 0xFFFF3B30, topLeft→bottomRight), `AppColors.aiGradient` (purple → magenta). All are `static const`.
- `app_spacing.dart`: `AppSpacing.xs=4, sm=8, md=12, lg=16, xl=24, xxl=32`; `radiusLg=16, radiusXl=24, radiusXxl=28`; `minTapTarget=48`. All `static const double`.
- `app_shadows.dart`: `AppShadows.card` is a `List<BoxShadow>` (NON-const — a mutable `static` field). `AppShadows.glow(Color color, {double strength = 0.35, double blur = 24})` returns `List<BoxShadow>` (also non-const). **Because these are non-const lists, any `BoxDecoration` whose `boxShadow` is set from them CANNOT be `const`.**
- Convention observed in `dashboard_screen.dart`: colors use `.withValues(alpha: x)` (not `.withOpacity`), gradient rings use `BoxShadow(..., spreadRadius: -2)`, and `AppColors` tokens are consumed directly. Match this.

### Design decision (chosen look, applied consistently)

Selected pill = `AppColors.primaryGradient` (orange→red energy) with glow `AppShadows.glow(AppColors.primary)`. Rationale: the pill represents the active "action" tab and the primary brand identity is energetic orange; using the orange glow (not the purple one) keeps the selected state coherent with CTAs across the app. The purple `electricPurple` glow option is explicitly NOT used, to avoid a split identity. This single choice is applied to whichever tab is selected (the AI tab included) so the bar reads consistently.

### Design decision (pill contains icon + label; unselected = icon only)

Selected tab renders a horizontal gradient pill holding the filled icon + label text (white). Unselected tabs render just the outlined icon in `AppColors.mutedGray` with no label, so the bar stays uncluttered and the selected tab is unmistakable. The pill width animates implicitly via `AnimatedContainer`, and the icon/label swap via the same rebuild. Rationale: an expanding gradient pill is the clearest premium "selected" affordance and matches the dashboard's gradient-surface language; keeping it subtle (200ms, easeOut) satisfies the "no gaudy animation" constraint.

---

## Byte-for-byte preserved regions (MUST NOT CHANGE)

Everything below stays verbatim. Only the `bottomNavigationBar:` value in `build` changes, plus NEW private helper widgets/methods appended to the file and NEW token imports added at the top.

1. All existing imports, in order:
   ```
   import 'package:flutter/material.dart';
   import 'package:flutter_riverpod/flutter_riverpod.dart';
   import '../ai/presentation/ai_coach_screen.dart';
   import '../alarm/providers/alarm_providers.dart';
   import '../games/presentation/morning_challenge_screen.dart';
   import '../home/presentation/dashboard_screen.dart';
   import '../notifications/notification_providers.dart';
   import '../nutrition/presentation/nutrition_screen.dart';
   import '../profile/presentation/profile_screen.dart';
   import '../workout/presentation/workout_hub_screen.dart';
   import 'shell_providers.dart';
   ```
   (NEW imports are ADDED after these — see item 1 below — existing lines are not reordered or removed.)

2. The class declarations and `_routingChallenge` flag:
   `class HomeShell extends ConsumerWidget { ... }` header + doc comment, `_HomeShellState` with `WidgetsBindingObserver`, `bool _routingChallenge = false;`.

3. The `static const _tabs = <Widget>[ DashboardScreen(), WorkoutHubScreen(), NutritionScreen(), AiCoachScreen(), ProfileScreen() ];` list — exact contents and order.

4. Lifecycle/startup methods, verbatim: `initState`, `dispose`, `didChangeAppLifecycleState`, `_initServices`, `_wireAlarmChallenge`, `_checkPendingChallenge`, `_routeToChallenge`.

5. Inside `build`: the first line `final index = ref.watch(homeTabProvider);`, the `Scaffold`, and `body: IndexedStack(index: index, children: _tabs),` — all unchanged. Selection must remain `ref.read(homeTabProvider.notifier).state = i`.

The 5 tabs keep labels `Home, Workout, Nutrition, AI, Profile` and icons `home / fitness_center / restaurant / auto_awesome / person` (outlined unselected, filled selected).

---

## Steps

- [ ] 1. Add design-system token imports to `home_shell.dart`.
      Add these three imports after the existing `import 'shell_providers.dart';` line (relative path from `features/shell/` to `core/theme/` is `../../core/theme/`):
      `import '../../core/theme/app_colors.dart';`
      `import '../../core/theme/app_spacing.dart';`
      `import '../../core/theme/app_shadows.dart';`
      Files: `frontend/lib/features/shell/home_shell.dart`
      Verify: `cd frontend && flutter analyze lib/features/shell/home_shell.dart` — no "unused import" errors remain after step 3 wires them in (expect transient unused-import warnings until step 3; final analyze in step 4 must be clean).

- [ ] 2. Define an immutable tab-descriptor list as a `static const` on `_HomeShellState`.
      Add a small private model class `_NavItem` (fields: `IconData icon`, `IconData selectedIcon`, `String label`; `const` constructor) at the bottom of the file, and a `static const List<_NavItem> _navItems` on `_HomeShellState` with the 5 entries in order: Home (`Icons.home_outlined` / `Icons.home`), Workout (`Icons.fitness_center_outlined` / `Icons.fitness_center`), Nutrition (`Icons.restaurant_outlined` / `Icons.restaurant`), AI (`Icons.auto_awesome_outlined` / `Icons.auto_awesome`), Profile (`Icons.person_outline` / `Icons.person`). This mirrors the current `NavigationDestination` data without touching `_tabs`.
      Files: `frontend/lib/features/shell/home_shell.dart`
      Verify: part of step 4 analyze; list length must equal 5 to match `_tabs`.

- [ ] 3. Replace the `bottomNavigationBar: NavigationBar(...)` value with the custom premium bar, and add the private builder widgets.
      In `build`, replace ONLY the `bottomNavigationBar:` argument (the whole `NavigationBar(...)` expression) with `bottomNavigationBar: _MidnightNavBar(selectedIndex: index, items: _navItems, onSelected: (i) => ref.read(homeTabProvider.notifier).state = i),`. Leave `body: IndexedStack(...)` and the surrounding `Scaffold` untouched.
      Then append two private `StatelessWidget`s at the bottom of the file:

      - `_MidnightNavBar` — the bar container. Structure:
        - `Container` with `decoration: BoxDecoration(color: AppColors.surface, border: Border(top: BorderSide(color: AppColors.divider, width: 1)), boxShadow: AppShadows.card)`. (NON-const because `AppShadows.card` is a runtime list.)
        - Child: `SafeArea(top: false, child: Padding(padding: const EdgeInsets.symmetric(horizontal: AppSpacing.sm, vertical: AppSpacing.sm), child: Row(children: [ ...for each item: Expanded(child: _NavTab(...)) ])))`. `SafeArea(top: false)` respects the bottom gesture inset; `Expanded` per tab gives responsive equal widths with no fixed-width overflow.
        - Each `_NavTab` gets `item`, `selected: i == selectedIndex`, and `onTap: () => onSelected(i)`.

      - `_NavTab` — one tab cell. Structure:
        - Root `GestureDetector`/`InkWell` wrapped so the full cell is tappable; enforce `ConstrainedBox(constraints: const BoxConstraints(minHeight: AppSpacing.minTapTarget))` (48 px min tap target). Prefer `InkWell` with `borderRadius: BorderRadius.circular(AppSpacing.radiusXl)` and `onTap`, inside a `Material(type: MaterialType.transparency)` so the ripple is contained.
        - Inside: `AnimatedContainer(duration: const Duration(milliseconds: 200), curve: Curves.easeOut, padding: EdgeInsets.symmetric(horizontal: selected ? AppSpacing.md : AppSpacing.sm, vertical: AppSpacing.sm), decoration: ...)`.
          - Selected decoration: `BoxDecoration(gradient: AppColors.primaryGradient, borderRadius: BorderRadius.circular(AppSpacing.radiusXl), boxShadow: AppShadows.glow(AppColors.primary))`.
          - Unselected decoration: `const BoxDecoration()` (transparent, no shadow).
        - Child: a `Row(mainAxisSize: MainAxisSize.min, mainAxisAlignment: MainAxisAlignment.center, children: [...])` containing:
          - `Icon(selected ? item.selectedIcon : item.icon, size: 22, color: selected ? AppColors.white : AppColors.mutedGray)`.
          - Label: wrap in `AnimatedSwitcher(duration: const Duration(milliseconds: 150), ...)` OR simply render the label only when `selected`, as `Padding(left: AppSpacing.xs) + Flexible(child: Text(item.label, maxLines: 1, overflow: TextOverflow.ellipsis, style: TextStyle(color: AppColors.white, fontWeight: FontWeight.w700, fontSize: 12, letterSpacing: 0.3)))`. Use `Flexible` + ellipsis so a long label never overflows the `Expanded` cell on small phones. When unselected, render no label (`SizedBox.shrink`).
      Files: `frontend/lib/features/shell/home_shell.dart`
      Verify: `cd frontend && flutter analyze` — clean (no errors, no warnings from the new code; the step-1 imports are now all used).

- [ ] 4. Full validation of the single-file change.
      Confirm the preserved regions (imports order, `_tabs`, all lifecycle methods, `IndexedStack` body, `ref.watch(homeTabProvider)` / `ref.read(homeTabProvider.notifier).state`) are byte-for-byte intact and only `bottomNavigationBar` + appended helpers changed.
      Files: `frontend/lib/features/shell/home_shell.dart`
      Verify: `cd frontend && flutter analyze` prints "No issues found!"; then `cd frontend && flutter test` runs the existing suite with no new failures attributable to this file. If the project has no shell-specific test, `flutter analyze` clean + a manual confirmation that the 5 tabs still switch (selected index drives `IndexedStack` and the gradient pill) is the acceptance bar.

---

## Notes / assumptions

- `const` caveat: because `AppShadows.card` and `AppShadows.glow(...)` return runtime (non-const) lists, the `_MidnightNavBar` container `BoxDecoration` and the `_NavTab` selected `BoxDecoration` must NOT be marked `const`. The analyzer will flag it otherwise; this is the one subtle correctness point.
- No new colors/constants are introduced — every color, radius, spacing, and shadow comes from the three token files.
- No routing, provider, or lifecycle behavior changes: selection still writes `homeTabProvider`, body still reads it via `IndexedStack`.
- Responsiveness: `Expanded` per tab + `Flexible`/ellipsis label + `SafeArea(top: false)` cover small/large phones and the bottom inset. Tap target >= 48 via `minHeight`.
- Animation stays subtle: a single 200ms `AnimatedContainer` for the pill expand + color/icon swap; no bounce, no infinite animation.
