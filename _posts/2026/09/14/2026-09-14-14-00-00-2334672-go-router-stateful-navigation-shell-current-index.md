---
layout: post
title: "go_router StatefulNavigationShell.currentIndex - Keep Flutter Tabs in Sync"
description: "Learn how StatefulNavigationShell.currentIndex drives Flutter tab selection, deep links, branch switching, and safe restoration in go_router."
date: 2026-09-14
tags: [go_router, navigation, state_management, Web]
comments: true
share: true
---

![go_router StatefulNavigationShell currentIndex keeping Flutter tabs synchronized](/assets/images/go-router-stateful-shell-route.png)

`StatefulNavigationShell.currentIndex` is the value that should drive a Flutter bottom navigation bar when `StatefulShellRoute` owns multiple navigation branches. The common mistake is to keep a second `_selectedTab` integer in the page. It starts at zero, but a deep link such as `/settings/profile` can open branch 1 while the UI still highlights branch 0.

## The problem with a second tab state

`StatefulShellRoute` keeps a separate `Navigator` for each branch. The shell can switch branches because of a tap, a deep link, browser back navigation, or state restoration. A manually maintained index only knows about the tap that changed it.

| Source of navigation | Manual `_selectedTab` | `currentIndex` |
| --- | --- | --- |
| User taps a tab | Usually correct | Correct |
| Deep link opens another branch | Stale | Correct |
| Browser back/forward | Easy to desynchronize | Correct |
| Restored tab stack | Must be reconstructed | Exposes shell state |

The shell is the owner of branch selection, so the widget should render that value directly.

## Build a branch-aware scaffold

The `builder` receives the `StatefulNavigationShell`. Keep it as the single source of truth: pass its index to the navigation UI and call `goBranch` when the user selects another tab.

Keep the shell as the single source of truth in the scaffold:

```dart
class AppScaffold extends StatelessWidget {
  const AppScaffold({required this.navigationShell, super.key});

  final StatefulNavigationShell navigationShell;

  void _onDestinationSelected(int index) {
    navigationShell.goBranch(
      index,
      initialLocation: index == navigationShell.currentIndex,
    );
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: navigationShell,
      bottomNavigationBar: NavigationBar(
        selectedIndex: navigationShell.currentIndex,
        onDestinationSelected: _onDestinationSelected,
        destinations: const [
          NavigationDestination(icon: Icon(Icons.home), label: 'Home'),
          NavigationDestination(icon: Icon(Icons.settings), label: 'Settings'),
        ],
      ),
    );
  }
}
```

The `initialLocation` choice here is deliberate. Tapping the already selected tab returns that branch to its initial route. Tapping a different tab keeps the branch's existing stack, so a user who was editing `/settings/profile` can leave and come back without losing that page.

## Deep links and restored state

When the app starts at `/settings/profile`, go_router selects the settings branch before the shell builds. `navigationShell.currentIndex` therefore becomes `1`, and `NavigationBar` highlights Settings without an extra redirect or `setState` call.

This also matters on Flutter web. The URL is the navigation input, while `currentIndex` is the presentation value. Do not infer the index by checking whether the URL starts with `/settings`; nested route groups, redirects, and encoded paths make that approach fragile.

For restoration, keep branch order and route identity stable. If Home moves from index 0 to index 1 in a later release, restored navigation data can point at a different branch even though the code still compiles.

## Traps I hit while testing

- Calling `goBranch(index, initialLocation: true)` for every tap erases nested branch history. Use it when the selected branch is tapped again or the product explicitly requires a reset.
- Calling `context.go` with a hard-coded tab path bypasses the shell's branch-preserving behavior. Use `goBranch` for tab changes.
- Changing branch order is a restoration-data change, not only a UI refactor.

## Quick checklist

1. Set `NavigationBar.selectedIndex` to `navigationShell.currentIndex`.
2. Call `navigationShell.goBranch` from tab callbacks.
3. Reset only the already selected branch when that is the intended UX.
4. Test a deep link, browser back/forward, and a restored nested route.

`currentIndex` looks like a small convenience property, but it prevents the shell and the tab bar from developing two different ideas about the active destination. Let go_router own that identity, and the surrounding UI becomes much easier to reason about.
