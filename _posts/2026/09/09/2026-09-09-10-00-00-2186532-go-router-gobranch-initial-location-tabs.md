---
layout: post
title: "go_router goBranch initialLocation - Reset Flutter Tabs Without Losing Navigation State"
description: "Learn how go_router goBranch initialLocation changes Flutter tab behavior, when to reset a branch, and how to test nested navigation safely."
date: 2026-09-09
tags: [go_router, navigation, state_management, testing, Flutter]
comments: true
share: true
---

![go_router StatefulShellRoute branch navigation and tab reset behavior](/assets/images/go-router-stateful-shell-route.png)

The `go_router` `goBranch` method has a small `initialLocation` switch that decides whether a Flutter tab returns to its branch root or restores the route the user left open. The default behavior preserves the branch stack. Passing `initialLocation: true` makes a repeated tab selection behave like a reset to that branch's initial location.

That distinction matters in a `StatefulShellRoute`. Each branch owns a separate `Navigator`, so a user can open `/orders/42`, switch to `/settings`, and later return to `/orders/42`. That is usually the right experience. A second tap on the already-selected Orders tab, however, often needs to take the user back to `/orders` instead of doing nothing.

## The two branch behaviors

| User action | `initialLocation` | Result |
| --- | --- | --- |
| Select another tab | `false` or omitted | Activates the branch and restores its previous stack |
| Select the current tab again | `false` | Keeps the current nested route |
| Select the current tab again | `true` | Navigates that branch to its initial location |

The option does not rebuild the whole shell and it does not clear every branch. It changes the navigation request for the branch being selected. That is why it works well for a common bottom-navigation convention: preserve state when switching tabs, pop to the tab root when tapping the active tab.

## A small StatefulShellRoute setup

The shell builder receives a `StatefulNavigationShell`. Keep that object as the owner of branch changes and pass the selected index from it to the navigation bar.

```dart
final _rootNavigatorKey = GlobalKey<NavigatorState>();
final _homeNavigatorKey = GlobalKey<NavigatorState>();
final _ordersNavigatorKey = GlobalKey<NavigatorState>();

final router = GoRouter(
  navigatorKey: _rootNavigatorKey,
  initialLocation: '/home',
  routes: [
    StatefulShellRoute.indexedStack(
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
          navigatorKey: _ordersNavigatorKey,
          routes: [
            GoRoute(
              path: '/orders',
              builder: (context, state) => const OrdersPage(),
              routes: [
                GoRoute(
                  path: ':id',
                  builder: (context, state) =>
                      OrderPage(id: state.pathParameters['id']!),
                ),
              ],
            ),
          ],
        ),
      ],
    ),
  ],
);
```

The route tree gives the Orders branch an initial location of `/orders`. Its nested page is `/orders/:id`, so the branch has a meaningful stack to preserve and a clear root to restore.

## Applying the reset policy at the tab bar

The shell does not need to inspect the current URL manually. `currentIndex` identifies the active branch, and `goBranch` handles the navigation operation.

```dart
class AppScaffold extends StatelessWidget {
  const AppScaffold({required this.navigationShell, super.key});

  final StatefulNavigationShell navigationShell;

  void _selectTab(int index) {
    final isCurrentTab = index == navigationShell.currentIndex;

    navigationShell.goBranch(
      index,
      initialLocation: isCurrentTab,
    );
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: navigationShell,
      bottomNavigationBar: NavigationBar(
        selectedIndex: navigationShell.currentIndex,
        onDestinationSelected: _selectTab,
        destinations: const [
          NavigationDestination(icon: Icon(Icons.home), label: 'Home'),
          NavigationDestination(icon: Icon(Icons.receipt), label: 'Orders'),
        ],
      ),
    );
  }
}
```

The important detail is that `initialLocation` is conditional. Passing `true` for every tap would reset a branch even when the user is coming back from another tab, which defeats the state-preserving purpose of `StatefulShellRoute`. Passing `true` only for the active tab gives the familiar behavior:

```text
/orders → /orders/42 → tap Home → tap Orders     = /orders/42
/orders/42 → tap Orders                           = /orders
```

The first line preserves the order detail page across a tab switch. The second line treats a repeated Orders tap as “go to the top of this section.”

## When a reset is the wrong choice

Not every tab should pop to its root. A mail app may want the Inbox tab to preserve a search result. A checkout flow may need to keep a partially completed form. In those cases, the tab bar should always call:

```dart
navigationShell.goBranch(index);
```

Use the reset policy only when the product language supports it. The API does not know whether a nested page contains unsaved work, a playback session, or a scroll position that users expect to survive.

There is also a distinction between resetting a branch and replacing application state. `initialLocation: true` moves the selected branch to its configured root; it does not clear providers, delete cached data, or recreate the shell. If the root screen still shows old data, that is a data-layer policy rather than a routing failure.

## A test that catches the real regression

A flat router test can pass while the shell behaves incorrectly. Exercise the nested route and the tab interaction together.

```dart
testWidgets('reselecting the active tab returns to its branch root',
    (tester) async {
  final router = createRouter(initialLocation: '/orders/42');

  await tester.pumpWidget(
    MaterialApp.router(routerConfig: router),
  );
  await tester.pumpAndSettle();

  expect(find.text('Order 42'), findsOneWidget);

  await tester.tap(find.byIcon(Icons.receipt));
  await tester.pumpAndSettle();

  expect(router.state.uri.path, '/orders');
  expect(find.text('Orders'), findsOneWidget);
});
```

Also test the opposite path: open `/orders/42`, select Home, then select Orders and assert that `/orders/42` is restored. That test proves the conditional policy instead of checking only the reset case.

The practical rule is simple: let branch switches preserve the independent Navigators, and use `initialLocation: true` only when the user reselects the active branch. This keeps deep navigation useful without making a familiar tab gesture feel inert.

Reference: [StatefulNavigationShell API](https://pub.dev/documentation/go_router/latest/go_router/StatefulNavigationShell-class.html) and [StatefulShellRoute API](https://pub.dev/documentation/go_router/latest/go_router/StatefulShellRoute-class.html).
