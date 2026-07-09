---
layout: post
title: "flutter_animate Drawer Navigation Motion - menu feedback without route noise"
description: "Use flutter_animate with Flutter Drawer and NavigationDrawer to add clear menu feedback without fighting gestures, routes, or selection state."
date: 2026-07-10
tags: [flutter_animate, navigation, animation, Android]
comments: true
share: true
---

![flutter_animate Drawer navigation motion](https://images.unsplash.com/photo-1516321318423-f06f85e504b3?w=800&q=80)

This image is a useful reminder that drawer motion should help people locate navigation choices, not turn the menu into a second page transition.

The best place to use `flutter_animate` with a Flutter `Drawer` is inside the drawer content, not around the drawer itself. Let `Scaffold`, `DrawerController`, gestures, scrims, focus trapping, and route timing stay boring. Animate the account header, the selected destination indicator, or a small status chip after a navigation action.

The first version I tried wrapped the entire `Drawer` in `.animate().slideX()`. It looked reasonable when opened with the hamburger button. Then edge-swipe opening felt late, because Flutter was already moving the drawer and my animation added a second horizontal movement. The menu appeared to chase the gesture instead of following it. That is route noise, not helpful feedback.

## Keep the drawer shell stable

A drawer is already an animated surface. It has an entrance, a barrier, keyboard handling, and platform-specific gesture behavior. `flutter_animate` should not replace that layer. The safer pattern is to keep the shell plain and animate children that communicate state.

```dart
Scaffold(
  appBar: AppBar(
    title: const Text('Billing'),
  ),
  drawer: Drawer(
    child: SafeArea(
      child: Column(
        children: [
          const AccountDrawerHeader(name: 'Colin Flutter'),
          Expanded(
            child: ListView(
              padding: EdgeInsets.zero,
              children: [
                DrawerDestinationTile(
                  icon: Icons.receipt_long,
                  label: 'Invoices',
                  selected: section == BillingSection.invoices,
                  onTap: () => openSection(BillingSection.invoices),
                ),
                DrawerDestinationTile(
                  icon: Icons.credit_card,
                  label: 'Payment methods',
                  selected: section == BillingSection.cards,
                  onTap: () => openSection(BillingSection.cards),
                ),
              ],
            ),
          ),
        ],
      ),
    ),
  ),
  body: BillingSectionView(section: section),
);
```

The drawer above has no custom entrance animation. That is intentional. The motion budget belongs to small details users can interpret after the drawer is already visible.

| Drawer event | Animate | Avoid |
| --- | --- | --- |
| Drawer opens | header content fade, 80-120 ms | custom slide on the whole drawer |
| Destination changes | selected icon or label color pulse | replay every menu item |
| Account switches | avatar scale and text fade | closing and reopening the drawer |
| Permission change | disabled row opacity update | removing rows with large motion |

## Animate selected state, not navigation

The selected destination is the useful signal. If a user opens the drawer from a deep screen, the menu should quickly show where they are. A short fade or color pulse on the selected row is enough.

```dart
class DrawerDestinationTile extends StatelessWidget {
  const DrawerDestinationTile({
    super.key,
    required this.icon,
    required this.label,
    required this.selected,
    required this.onTap,
  });

  final IconData icon;
  final String label;
  final bool selected;
  final VoidCallback onTap;

  @override
  Widget build(BuildContext context) {
    final colorScheme = Theme.of(context).colorScheme;

    return ListTile(
      leading: Icon(
        icon,
        color: selected ? colorScheme.primary : null,
      ),
      title: Text(
        label,
        style: TextStyle(
          fontWeight: selected ? FontWeight.w600 : FontWeight.w400,
        ),
      ),
      selected: selected,
      onTap: onTap,
    )
        .animate(
          key: ValueKey('$label-$selected'),
          target: selected ? 1 : 0,
        )
        .fade(begin: 0.88, end: 1, duration: 120.ms)
        .scale(
          begin: const Offset(0.98, 0.98),
          end: const Offset(1, 1),
          duration: 120.ms,
        );
  }
}
```

The `ValueKey` is doing real work here. It changes when the selected state changes, not when the drawer rebuilds because the theme, account object, or parent `Scaffold` updates. Without that boundary, a drawer with 8 destinations can replay every row after one tap.

I also avoid horizontal movement on drawer rows. The drawer itself moves horizontally. If the selected row slides too, the animation reads like a broken continuation of the drawer gesture. A tiny scale and opacity change is easier to parse.

## Close timing still matters

Navigation drawers often close immediately after a tap. If the destination changes and the drawer closes in the same frame, the row animation may not be visible. That does not mean the answer is a 500 ms delay. Most of the time the destination page should win.

For same-route section changes, I let the selected row animate and close the drawer after a short delay:

```dart
Future<void> openSection(BillingSection next) async {
  setState(() => section = next);
  await Future<void>.delayed(120.ms);
  if (!mounted) return;
  Navigator.of(context).pop();
}
```

For real route navigation, I usually close the drawer immediately and animate the destination page content instead. Delaying route navigation for a drawer flourish makes the app feel slower, especially on phones where the drawer is already a temporary surface.

## NavigationDrawer has the same rule

Material 3 `NavigationDrawer` gives you `NavigationDrawerDestination` and `selectedIndex`, but the rule does not change. Keep `NavigationDrawer` in charge of selection semantics and animate the content inside labels or icons only when the state changes.

```dart
NavigationDrawer(
  selectedIndex: selectedIndex,
  onDestinationSelected: changeDestination,
  children: destinations.map((destination) {
    final selected = destination.index == selectedIndex;

    return NavigationDrawerDestination(
      icon: Icon(destination.icon)
          .animate(target: selected ? 1 : 0)
          .scaleXY(begin: 1, end: 1.08, duration: 100.ms),
      label: Text(destination.label),
    );
  }).toList(),
);
```

One trap: do not animate `selectedIndex` by creating a fake intermediate index. The drawer selection is app state, not animation state. If analytics, restoration, or deep links read that value, a fake transition index can create bugs that are hard to reproduce.

## Short checklist

- Keep `Drawer` and `NavigationDrawer` as the gesture and semantics owners.
- Animate selected destination details, not the whole drawer surface.
- Use keys tied to destination identity and selected state.
- Prefer fade, scale, and color cues over horizontal row movement.
- Delay drawer closing only for same-route state changes where the feedback is useful.

`flutter_animate` fits drawer navigation when it stays below the route layer. The drawer opens normally, the selected item becomes easier to read, and route changes remain fast enough to feel like navigation instead of decoration.
