---
layout: post
title: "go_router navigatorContainerBuilder - Animate Stateful Flutter Tabs Without Losing State"
description: "Learn how go_router navigatorContainerBuilder controls StatefulShellRoute tab transitions while preserving each Flutter branch navigator and its navigation stack."
date: 2026-08-29
tags: [go_router, navigation, animation, Flutter, performance]
comments: true
share: true
---

![go_router navigatorContainerBuilder animating stateful Flutter tab navigators](/assets/images/go-router-stateful-shell-route.png)

`StatefulShellRoute.indexedStack` preserves each Flutter tab's navigation stack, but it does not animate the switch between branches. The `navigatorContainerBuilder` parameter is the place to add that motion without replacing the branch Navigators or losing their state.

## The problem with treating tabs like ordinary routes

I initially tried to solve tab motion inside each `GoRoute` with a `CustomTransitionPage`. That animated `/home/details` nicely, but it did nothing when the user tapped the Profile tab. Those are two different transitions:

| Transition | Controlled by | Typical API |
| --- | --- | --- |
| Home list → Home details | The active branch Navigator | `pageBuilder` / `CustomTransitionPage` |
| Home tab → Profile tab | The shell's branch container | `navigatorContainerBuilder` |

The shell owns parallel Navigators. It needs a container that can display all of them while deciding which branch is visible and interactive.

## Keep the default route structure

The route configuration stays almost identical to an ordinary `StatefulShellRoute`. The only meaningful change is replacing `.indexedStack` with the full constructor and supplying a container builder.

Here is a small two-tab configuration. The route keys are important because each branch owns an independent Navigator.

```dart
final _rootNavigatorKey = GlobalKey<NavigatorState>();
final _homeNavigatorKey = GlobalKey<NavigatorState>();
final _settingsNavigatorKey = GlobalKey<NavigatorState>();

final router = GoRouter(
  navigatorKey: _rootNavigatorKey,
  routes: [
    StatefulShellRoute(
      navigatorContainerBuilder: (context, navigationShell, children) {
        return AnimatedBranchContainer(
          currentIndex: navigationShell.currentIndex,
          children: children,
        );
      },
      builder: (context, state, navigationShell) {
        return AppScaffold(navigationShell: navigationShell);
      },
      branches: [
        StatefulShellBranch(
          navigatorKey: _homeNavigatorKey,
          routes: [
            GoRoute(
              path: '/home',
              builder: (context, state) => const HomePage(),
            ),
          ],
        ),
        StatefulShellBranch(
          navigatorKey: _settingsNavigatorKey,
          routes: [
            GoRoute(
              path: '/settings',
              builder: (context, state) => const SettingsPage(),
            ),
          ],
        ),
      ],
    ),
  ],
);
```

The `children` list contains the already-built branch Navigator widgets. Do not replace it with new pages from `currentIndex`; doing so bypasses the navigation state that `StatefulShellRoute` is maintaining.

## A container that fades between branches

For a first implementation, a cross-fade is easier to reason about than a custom `PageView`. Every branch stays in the widget tree, while inactive branches stop receiving input and ticking animations.

```dart
class AnimatedBranchContainer extends StatelessWidget {
  const AnimatedBranchContainer({
    required this.currentIndex,
    required this.children,
    super.key,
  });

  final int currentIndex;
  final List<Widget> children;

  @override
  Widget build(BuildContext context) {
    return Stack(
      fit: StackFit.expand,
      children: [
        for (var index = 0; index < children.length; index++)
          IgnorePointer(
            ignoring: index != currentIndex,
            child: TickerMode(
              enabled: index == currentIndex,
              child: AnimatedOpacity(
                opacity: index == currentIndex ? 1 : 0,
                duration: const Duration(milliseconds: 220),
                curve: Curves.easeOutCubic,
                child: KeyedSubtree(
                  key: ValueKey(index),
                  child: children[index],
                ),
              ),
            ),
          ),
      ],
    );
  }
}
```

The `ValueKey(index)` is not cosmetic. It gives Flutter a stable identity for each branch when the list is rebuilt. Without stable identity, a more complicated container can accidentally make a Navigator look like a new widget and reset visible state.

For a slide transition, replace `AnimatedOpacity` with an `AnimatedSlide` around each branch. Keep the `Stack` and `IgnorePointer` structure. A `PageView` can also work, but it introduces a second page-position model that must stay synchronized with `navigationShell.goBranch(index: ...)` and browser deep links.

## Wire the bottom navigation to the shell

The shell, rather than the container, remains responsible for changing the active branch. This keeps the URL and the selected tab in agreement.

```dart
class AppScaffold extends StatelessWidget {
  const AppScaffold({required this.navigationShell, super.key});

  final StatefulNavigationShell navigationShell;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: navigationShell,
      bottomNavigationBar: NavigationBar(
        selectedIndex: navigationShell.currentIndex,
        onDestinationSelected: (index) {
          navigationShell.goBranch(
            index: index,
            initialLocation: index == navigationShell.currentIndex,
          );
        },
        destinations: const [
          NavigationDestination(icon: Icon(Icons.home), label: 'Home'),
          NavigationDestination(
            icon: Icon(Icons.settings),
            label: 'Settings',
          ),
        ],
      ),
    );
  }
}
```

`initialLocation` deserves a deliberate choice. Passing `true` when the current tab is tapped returns that branch to its initial route. If the expected behavior is “tap the current tab and keep its detail page,” omit the argument or pass `false`. I have seen this mistaken for a container animation bug because the fade worked while the branch unexpectedly popped back to its root.

## Traps worth testing

Test the shell with a real nested route, not only two root pages. Push a detail page in Home, switch to Settings, and return to Home. The detail page should still be present. Then test a deep link such as `/home/details` in a fresh browser tab; the selected index should come from the router, not from a locally initialized integer.

Also check these boundaries:

| Check | Failure symptom |
| --- | --- |
| Inactive branch ignores pointers | An invisible tab handles taps |
| Inactive branch uses `TickerMode(false)` | Hidden animations consume battery or CPU |
| Children retain stable keys | A tab unexpectedly rebuilds its Navigator |
| Container does not call `goBranch` | URL and navigation state fight each other |

`navigatorContainerBuilder` is a rendering boundary, not a second router. Let `StatefulNavigationShell` change branches, and let the container decide how those existing branch Navigators appear. That separation gives you animated tab switches while keeping the state-preserving behavior that made `StatefulShellRoute` useful in the first place.

Reference: [StatefulShellRoute API](https://pub.dev/documentation/go_router/latest/go_router/StatefulShellRoute-class.html) and [ShellNavigationContainerBuilder API](https://pub.dev/documentation/go_router/latest/go_router/ShellNavigationContainerBuilder.html).
