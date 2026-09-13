---
layout: post
title: "go_router StatefulShellBranch restorationScopeId - Restore Flutter Tab State Safely"
description: "Learn how StatefulShellBranch restorationScopeId separates Flutter tab restoration data and how to keep branch IDs stable across app releases."
date: 2026-09-13
tags: [go_router, navigation, state_management, Android, iOS]
comments: true
share: true
---

![go_router StatefulShellBranch restoring independent Flutter tab navigation state](/assets/images/go-router-stateful-shell-route.png)

`StatefulShellRoute` preserves each Flutter tab's navigation stack while the process is alive. That is only the in-memory part of the story. When Android or iOS recreates the app, each `StatefulShellBranch` also needs a stable `restorationScopeId` if its nested route state should come back independently. The useful boundary is one ID per branch, not one generated ID for the whole shell.

## What a branch restoration scope owns

| Scope | Restored state | Typical mistake |
| --- | --- | --- |
| `MaterialApp.restorationScopeId` | App-wide restoration bucket | Assuming it restores every page automatically |
| `StatefulShellRoute.restorationScopeId` | Shell-level navigation state | Treating it as durable business data |
| `StatefulShellBranch.restorationScopeId` | One tab's nested navigator state | Reusing the same ID for multiple branches |

The branch scope matters when a user opens `/orders/42`, switches to Settings, and the operating system later kills the process. A stable Orders scope gives that branch a place to restore its selected route and restorable widget state. It does not save an API response, a repository cache, or an authentication token.

## Configure IDs beside the branch definition

Keep the IDs close to the route tree so they are reviewed when a tab is renamed or moved. The IDs must be unique within the restoration hierarchy and must not be generated during `build()`.

```dart
final rootNavigatorKey = GlobalKey<NavigatorState>();

final router = GoRouter(
  navigatorKey: rootNavigatorKey,
  restorationScopeId: 'app_router',
  routes: [
    StatefulShellRoute.indexedStack(
      restorationScopeId: 'main_shell',
      builder: (context, state, navigationShell) {
        return AppScaffold(navigationShell: navigationShell);
      },
      branches: [
        StatefulShellBranch(
          restorationScopeId: 'orders_branch',
          routes: [
            GoRoute(
              path: '/orders',
              builder: (context, state) => const OrdersPage(),
              routes: [
                GoRoute(
                  path: ':orderId',
                  builder: (context, state) => OrderPage(
                    orderId: state.pathParameters['orderId']!,
                  ),
                ),
              ],
            ),
          ],
        ),
        StatefulShellBranch(
          restorationScopeId: 'settings_branch',
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

The IDs describe restoration identity, not the current URL. Changing `orders_branch` to `shop_branch` may make previously stored state unavailable because the platform sees a different bucket. I treat such a rename like a small migration decision: preserve the old ID when continuity matters, or intentionally change it when the route structure is no longer compatible.

## Restoration is not persistence

There are three separate failure cases that look similar during testing:

| Symptom after process death | Likely boundary | Fix |
| --- | --- | --- |
| The selected tab resets | Shell or branch scope missing | Add stable restoration IDs |
| The detail route returns but data is empty | Repository data was memory-only | Reload by `orderId` from the route |
| A restored route no longer matches | Path or branch structure changed | Keep a compatible route or migrate the location |

For a restorable detail page, store the identity in the URL and reconstruct the model from that identity. A route such as `/orders/42` is recoverable; passing only an in-memory `Order` through `extra` is not enough for a cold restore. The router can restore the location, while the repository restores the data.

## Traps with shell branches

Do not assign the same `restorationScopeId` to `orders_branch` and `settings_branch`. Their navigators represent different state domains, and a shared bucket makes the stored data ambiguous. Also avoid testing only by switching tabs: that verifies `StatefulShellRoute`'s live stack preservation, not platform state restoration. Test the actual lifecycle by backgrounding or terminating the app, then reopening it with the same restoration scope.

On Flutter Web, the browser URL remains the primary source for a deep link and refresh. Branch restoration can complement that behavior, but it should not replace durable paths. Keep route IDs, query parameters, and branch structure stable enough that an old URL can still reconstruct the intended screen.

The practical rule is compact: give the app, shell, and each stateful branch deliberate restoration scopes; keep those IDs stable; put resource identity in route parameters; and load durable data outside the router. `StatefulShellBranch.restorationScopeId` then does one job well—separating tab navigation state across a process boundary without pretending to be a database.

References: [StatefulShellBranch API](https://pub.dev/documentation/go_router/latest/go_router/StatefulShellBranch-class.html), [StatefulShellRoute API](https://pub.dev/documentation/go_router/latest/go_router/StatefulShellRoute-class.html), and [Flutter state restoration](https://docs.flutter.dev/ui/adaptive-responsive/general).
