# Activity tab filter chips

## Goal

Add three filter chips (All, Income, Spending) under the "Recent activity" header of the Activity tab. Picking one filters the list below with a fade + slide + height collapse per row, and the selected chip animates its colours. The weekly total card and the bar chart do not change.

## Context

Project root: the app folder. Flutter 3.47.5, Dart SDK ^3.13.4. Not a git repository, so there is nothing to branch, commit or push. The `test/` folder is empty.

`lib/features/vault/presentation/pages/activity_tab.dart` (221 lines, the only Dart file to edit):
- L14 `_week`, L16-53 `_activity`: a const list of records `(icon, merchant, detail, amount)`. Five negative amounts, one positive (Salary, 4200.00). No zero amounts.
- L64-88 `_ActivityTabState` with `SingleTickerProviderStateMixin` and one `AnimationController _intro` (1300 ms), created in `initState`, replayed in `didUpdateWidget` when the tab becomes active, disposed in `dispose`.
- L100-185 a `ListView` with children: title (`Stagger` index 0), the week card (index 1, L114-165), "Recent activity" header (index 3, L167-171), `SizedBox(height: AppSpacing.sm)` (L172), then L173-183 a `for` loop that wraps each `ActivityTile` in `Stagger(index: 4 + i)`.
- L190-221 `_Bar`, a private widget in the same file. Follow that pattern for the new private widgets.

Hosting: `lib/features/vault/presentation/pages/home_shell.dart` L116 keeps all three tabs alive in a `Stack` with `TickerMode`, so the chosen filter survives tab switches. That is fine, leave it.

Widgets to reuse:
- `lib/core/widgets/pressable.dart`: `Pressable({required VoidCallback onTap, required Widget child})`, press scale feedback. Use it for the chip tap.
- `lib/core/widgets/stagger.dart`: `Stagger({animation, index, child})`, intro fade + rise. Start offset is `index * 0.09` clamped to 0.6, so higher indices are safe.
- `lib/features/vault/presentation/widgets/activity_tile.dart`: `ActivityTile`, stateless, no changes needed.

Existing chip-like pattern to mirror: `lib/features/vault/presentation/widgets/limit_sheet.dart` L90-112 (`Pressable` > `AnimatedContainer` 220 ms, `AppRadius.pill`, `AppTextStyles.caption.copyWith(fontWeight: FontWeight.w700)`). Selected colours across the app are `AppColors.accent` with `AppColors.onAccent` content (`lib/core/widgets/floating_nav.dart` L41-74).

Tokens (all exist, add none):
- `AppColors.accent`, `onAccent`, `surface`, `stroke`, `textSecondary` (`lib/core/constants/app_colors.dart`)
- `AppSpacing.xs 4`, `sm 8`, `md 12`, `lg 16`, `xl 24` (`app_spacing.dart`)
- `AppRadius.pill` (`app_radius.dart` L8). Do not use `AppRadius.chip`, that is the 6.r radius of the card's EMV chip and the bars.
- `AppTextStyles.caption` (12.sp, w500, textSecondary)

Localisation: `l10n.yaml` points at `lib/l10n`, template `app_en.arb`, `nullable-getter: false`. Generated files `lib/l10n/app_localizations*.dart` are checked in next to the ARB files. Activity keys today: `recentActivity` (en L50, ar L22), `activityTitle`, `activityWeek` (en L103-104, ar L51-52). No filter keys exist.

## Steps

1. `lib/l10n/app_en.arb`: after `"activityWeek": "This week",` add
   ```json
   "filterAll": "All",
   "filterIncome": "Income",
   "filterSpending": "Spending",
   ```

2. `lib/l10n/app_ar.arb`: after `"activityWeek": "هذا الأسبوع",` add
   ```json
   "filterAll": "الكل",
   "filterIncome": "الدخل",
   "filterSpending": "المصروفات",
   ```

3. From the project root run `flutter gen-l10n`. It rewrites `lib/l10n/app_localizations.dart`, `app_localizations_en.dart`, `app_localizations_ar.dart`. Do not hand-edit those.

4. `lib/features/vault/presentation/pages/activity_tab.dart`, top level, below `_activity`:
   - Add `import '../../../../core/widgets/pressable.dart';` in alphabetical position (between `app_text_styles.dart` and `rolling_number.dart`).
   - Add a private enum with the match rule:
     ```dart
     enum _Filter {
       all,
       income,
       spending;

       bool shows(double amount) => switch (this) {
         all => true,
         income => amount > 0,
         spending => amount < 0,
       };
     }
     ```
   - Add one shared duration: `const _swap = Duration(milliseconds: 320);`

5. Same file, `_ActivityTabState`:
   - Add field `var _filter = _Filter.all;`. No new controller, so `dispose` stays as is.
   - In `build`, after `l10n`, map labels with an exhaustive switch inside the chip loop (step 6). Do not store labels in state.

6. Same file, `ListView.children`, replace L172-183 (the `SizedBox(height: AppSpacing.sm)` and the tile loop) with:
   - `SizedBox(height: AppSpacing.md)`
   - `Stagger(animation: _intro, index: 4, child: Wrap(spacing: AppSpacing.sm, runSpacing: AppSpacing.sm, children: [...]))` where the children are one `_FilterChip` per `_Filter.values`:
     ```dart
     for (final f in _Filter.values)
       _FilterChip(
         label: switch (f) {
           _Filter.all => l10n.filterAll,
           _Filter.income => l10n.filterIncome,
           _Filter.spending => l10n.filterSpending,
         },
         selected: f == _filter,
         onTap: () => setState(() => _filter = f),
       ),
     ```
     `Wrap` follows `Directionality`, so chips start from the right in Arabic and wrap instead of overflowing at large text scales.
   - `SizedBox(height: AppSpacing.sm)`
   - The tile loop, now with index `5 + i` and a `_Reveal` between `Stagger` and `ActivityTile`:
     ```dart
     for (var i = 0; i < _activity.length; i++)
       Stagger(
         animation: _intro,
         index: 5 + i,
         child: _Reveal(
           visible: _filter.shows(_activity[i].amount),
           child: ActivityTile(...same four args as today...),
         ),
       ),
     ```
   Every row stays in the tree in a fixed position, so rows that survive a filter change do not rebuild or jump, and rows that leave or enter animate on their own.
   Leave everything above the "Recent activity" header (title, week card, total, bars) untouched. `total` and `peak` keep reading `_week`.

7. Same file, add two private stateless widgets next to `_Bar`:

   `_FilterChip({required String label, required bool selected, required VoidCallback onTap})`:
   ```dart
   Semantics(
     button: true,
     selected: selected,
     child: Pressable(
       onTap: onTap,
       child: AnimatedContainer(
         duration: _swap,
         curve: Curves.easeOutCubic,
         padding: EdgeInsets.symmetric(
           horizontal: AppSpacing.lg,
           vertical: AppSpacing.sm,
         ),
         decoration: BoxDecoration(
           color: selected ? AppColors.accent : AppColors.surface,
           borderRadius: AppRadius.pill,
           border: Border.all(
             color: selected ? AppColors.accent : AppColors.stroke,
           ),
         ),
         child: AnimatedDefaultTextStyle(
           duration: _swap,
           style: AppTextStyles.caption.copyWith(
             color: selected ? AppColors.onAccent : AppColors.textSecondary,
             fontWeight: FontWeight.w700,
           ),
           child: Text(label),
         ),
       ),
     ),
   )
   ```

   `_Reveal({required bool visible, required Widget child})`:
   ```dart
   TweenAnimationBuilder<double>(
     tween: Tween<double>(end: visible ? 1 : 0),
     duration: _swap,
     curve: Curves.easeOutCubic,
     child: child,
     builder: (context, t, child) {
       if (t == 0) return const SizedBox.shrink();
       return ClipRect(
         child: Align(
           alignment: AlignmentDirectional.topStart,
           heightFactor: t,
           child: Opacity(
             opacity: t,
             child: Transform.translate(
               offset: Offset(0, (t - 1) * AppSpacing.md),
               child: child,
             ),
           ),
         ),
       );
     },
   )
   ```
   Notes that matter:
   - A `Tween` with only `end` starts at `end`, so nothing animates on first build and the intro `Stagger` still owns the entrance.
   - Keep the curve at `easeOutCubic`. An overshooting curve (`easeOutBack`) pushes `t` outside 0..1 and trips the asserts in `Opacity` and `Align.heightFactor`.
   - The `t == 0` branch drops hidden rows from layout, paint and semantics once collapsed.
   - `Align` inside the `ListView` gets a tight width, so the tile `Row` keeps its full width.

8. Run `dart format lib`, then `flutter analyze`. Fix anything reported.

## Rules

- Tokens only: no hex colours, no raw pixel numbers in widgets. Sizes go through `AppSpacing` / `AppRadius` or `.w .h .r .sp`.
- Every user-facing string comes from `AppLocalizations`. Both ARB files get the same keys.
- RTL-safe: `EdgeInsetsDirectional`, `AlignmentDirectional`, `PositionedDirectional`. `EdgeInsets.symmetric` and `EdgeInsets.all` are direction-neutral and already used in this file.
- No new packages. No Cubit, Bloc or Provider for this: plain `setState` in `_ActivityTabState`.
- No narrating comments, no TODOs, no dead code. Zero comments are needed for this change.
- No new controllers are planned. If you add one anyway, create it in `initState` and dispose it.
- Do not touch `activity_tile.dart`, `stagger.dart`, `pressable.dart`, `home_shell.dart` or the constants files.
- Do not add tests or an empty state. The sample data always has rows for each filter.
- Do not run the app.

## Verify

From the project root:

1. `flutter gen-l10n` exits 0, then `grep -n "filterAll\|filterIncome\|filterSpending" lib/l10n/app_localizations.dart lib/l10n/app_localizations_en.dart lib/l10n/app_localizations_ar.dart` shows all three getters in each file.
2. `dart format lib` then `dart format --output=none --set-exit-if-changed lib` exits 0.
3. `flutter analyze` prints "No issues found!".
4. `grep -nE "0x[0-9A-Fa-f]{6,8}|EdgeInsets\.only|Alignment\.(center|top|bottom)(Left|Right)|//" lib/features/vault/presentation/pages/activity_tab.dart` returns nothing.
5. Read the diff of `activity_tab.dart`: lines 108-166 of the original (title and week card) are unchanged, and the only string literals in the new code are import paths.
6. Logic check by reading: All shows 6 rows, Income shows 1 (Salary), Spending shows 5.

## Open questions

None. Two choices made that are easy to flip if the look is off: the selected chip uses `AppColors.accent` with `onAccent` text (same as the floating nav), and the chip height is about 32 logical px (`AppSpacing.sm` vertical padding); swap to `AppSpacing.md` for a taller tap target.
