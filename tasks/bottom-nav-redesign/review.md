# Midnight Energy bottom navigation bar for HomeShell

HomeShell's Material `NavigationBar` is replaced with a custom dark bottom bar built from the Midnight Energy tokens. The selected tab expands into an orange→red gradient pill (filled icon + white label) with a warm `AppShadows.glow(AppColors.primary)`; unselected tabs are muted gray, icon-only. The five tabs, their order, labels, icons, and the `homeTabProvider` read/write wiring are intact, and every startup/lifecycle method is present and unmodified in behavior. The change is presentation-only and sits entirely in `home_shell.dart`.

Watch for: the commit records this file as `new file mode 100644` with 262 insertions and zero deletions, so git holds no prior tracked version to diff against — "byte-for-byte preserved" cannot be proven from history, only judged against the current file contents (confirmed). Icon size (`22`), label `fontSize` (`12`), and `letterSpacing` (`0.3`) are inline literals rather than tokens (confirmed), but they are typography/sizing, not color or radius, so they fall outside the stated token constraint. No blocking concerns.

**Verdict**: APPROVED

## High-level view

The lifecycle and wiring surface is fully preserved. `initState`/`dispose`/`didChangeAppLifecycleState`, the `WidgetsBindingObserver` mixin, `_initServices` (FCM init + `rescheduleReminders`), `_wireAlarmChallenge`, `_checkPendingChallenge`, `_routeToChallenge`, and the `_routingChallenge` guard all read exactly as the behavioral contract requires. The body is still `IndexedStack(index: index, children: _tabs)` with the five screens in the mandated order, and tab selection still reads `ref.watch(homeTabProvider)` and writes `ref.read(homeTabProvider.notifier).state = i`.

The presentation swap is clean. Only the `bottomNavigationBar` slot changed — from a Material `NavigationBar` to a private `_MidnightNavBar` plus `_NavTab` and a `_NavItem` descriptor list that mirrors `_tabs` 1:1. Labels (Home, Workout, Nutrition, AI, Profile) and icon pairs (home, fitness_center, restaurant, auto_awesome, person — outlined when unselected, filled when selected) match the spec.

Theming draws only on real tokens. `AppColors.surface`/`divider`/`primary`/`primaryGradient`/`white`/`mutedGray`, `AppSpacing.sm`/`md`/`xs`/`radiusXl`/`minTapTarget`, and `AppShadows.card`/`glow` all exist and are used as intended; no invented color or radius constants appear. The only inline literals are icon size and label font metrics, which are outside the color/radius constraint.

Responsiveness and accessibility hold up. Each tab is wrapped in `Expanded` (no fixed widths that could overflow), the tap target is floored at `AppSpacing.minTapTarget` (48) via a `ConstrainedBox`, the label uses `Flexible` + `maxLines: 1` + `TextOverflow.ellipsis`, and the bar respects the bottom inset with `SafeArea(top: false)`.

<details>
<summary>Issues (1)</summary>

1. **Inline sizing/typography literals** — `size: 22`, `fontSize: 12`, and `letterSpacing: 0.3` are hardcoded in `_NavTab` rather than drawn from a token or the text theme. Non-blocking (outside the color/radius constraint); consider sourcing the label from a theme text style for consistency.

</details>

<details>
<summary>Details</summary>

### Lifecycle and startup logic preserved

Every method named in the contract is present with unchanged behavior. `initState` adds the observer and schedules a post-frame callback that runs `_initServices()`, `_wireAlarmChallenge()`, then awaits `_checkPendingChallenge()`. `_initServices` still initializes `fcmServiceProvider` and calls `rescheduleReminders(ref)`, each wrapped in its own best-effort try/catch so a failure in one does not block the other or the app. `_wireAlarmChallenge` installs the `onChallengePending` callback that stashes the alarm id into `pendingChallengeProvider` and routes. `_checkPendingChallenge` queries `getPendingChallenge()` and routes when the id is non-negative. `_routeToChallenge` keeps the `_routingChallenge` reentrancy guard, resolves the alarm's difficulty and preferred games, checks `mounted` before navigating, and clears the guard on every exit path. `didChangeAppLifecycleState` re-checks for a pending challenge on resume. The `WidgetsBindingObserver` mixin and `dispose` observer removal are intact (confirmed).

Because git has no prior tracked revision of this file (the commit is a fresh add), the "byte-for-byte" claim is unverifiable from history. Judged against the current contents, the logic matches the required contract and shows no signs of accidental edits.

### Presentation swap confined to the nav bar

The `build` method's only structural change versus the described baseline is the `bottomNavigationBar` argument, now a `_MidnightNavBar`. The body remains `IndexedStack(index: index, children: _tabs)` with `_tabs` holding `DashboardScreen`, `WorkoutHubScreen`, `NutritionScreen`, `AiCoachScreen`, `ProfileScreen` in order. `_navItems` is a parallel const list whose comment explicitly notes the 1:1 mapping to `_tabs`, and its icon pairs and labels match the spec (outlined unselected, filled selected). Selection still flows through `ref.read(homeTabProvider.notifier).state = i`, and the selected index still comes from `ref.watch(homeTabProvider)` (confirmed).

### Token usage and the gradient pill

```dart
decoration: selected
    ? BoxDecoration(
        gradient: AppColors.primaryGradient,
        borderRadius: BorderRadius.circular(AppSpacing.radiusXl),
        boxShadow: AppShadows.glow(AppColors.primary),
      )
    : const BoxDecoration(),
```

Every color and radius reference resolves to a real token: `AppColors.surface` (bar fill), `AppColors.divider` (top hairline), `AppColors.primaryGradient` + `AppColors.primary` glow (selected pill), `AppColors.white` / `AppColors.mutedGray` (icon/label), `AppSpacing.radiusXl` (pill and ink radius), and `AppShadows.card` / `AppShadows.glow`. All were confirmed to exist in the theme files. The glow color is derived from `AppColors.primary`, so the pill and its halo stay coherent.

The inline literals `size: 22`, `fontSize: 12`, and `letterSpacing: 0.3` are typography and icon sizing, outside the color/radius constraint; the label could instead be sourced from the text theme.

### Responsiveness and tap targets

Each tab is an `Expanded` child of a `Row`, so the five cells share width evenly with no fixed widths to overflow on narrow phones. The selected pill grows its horizontal padding (`AppSpacing.md` vs `sm`) and reveals the label inside a `Flexible` with `maxLines: 1` and `TextOverflow.ellipsis`, so a long label clips rather than overflowing. `ConstrainedBox(minHeight: AppSpacing.minTapTarget)` floors the tap target at 48, and `SafeArea(top: false)` keeps the bar clear of the system gesture inset. The 200ms `AnimatedContainer` on the pill is the kind of subtle transition the brief asked for.

</details>

<details>
<summary>File map</summary>

- `lib/features/shell/home_shell.dart` — replaced the Material `NavigationBar` with the private `_MidnightNavBar`/`_NavTab`/`_NavItem` bottom bar; added theme imports (`app_colors`, `app_spacing`, `app_shadows`); lifecycle/wiring untouched.

Full diff: `git show fe21590 -- lib/features/shell/home_shell.dart` (committed as a new tracked file, 262 insertions).

</details>
