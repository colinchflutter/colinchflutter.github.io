---
layout: post
title: "go_router StatefulShellBranch.preload - Remove First-Tab Loading Jank in Flutter"
description: "Learn how go_router StatefulShellBranch.preload eagerly prepares Flutter tab branches, when it improves UX, and how to avoid unnecessary startup work."
date: 2026-08-31
tags: [go_router, navigation, performance, Flutter]
comments: true
share: true
---

![go_router StatefulShellBranch preload preparing Flutter tab navigators](/assets/images/go-router-stateful-shell-route.png)

`StatefulShellBranch.preload` makes a Flutter tab branch load before the user opens it for the first time. It is useful when the second tab contains a predictable, lightweight screen and the first tap currently shows a visible delay. It is not a general “preload everything” switch: enabling it for every branch can move the same work into app startup and make the initial frame slower.

## The problem with lazy tab branches

`StatefulShellRoute` creates a separate `Navigator` for each `StatefulShellBranch`. In practice, the first branch is visible immediately, while other branches may not build until `navigationShell.goBranch()` selects them. That lazy behavior saves startup work, but it can make a tab with expensive providers, a database query, or a large widget tree feel different from the already-open tab.

| Branch choice | First app frame | First visit to the tab | Good fit |
|---|---|---|---|
| `preload: false` | Lighter | May do setup on tap | Heavy or rarely visited areas |
| `preload: true` | More work up front | Usually smoother | Small, frequently used tabs |

The key distinction is timing. `preload` changes when the branch's initial location is loaded; it does not cache API responses, keep every screen in memory forever, or replace application-level state management.

## Enable it on selected branches

The following route keeps the home branch lazy but prepares the compact notifications branch while the shell is first entered:

```dart
final router = GoRouter(
  initialLocation: '/home',
  routes: [
    StatefulShellRoute.indexedStack(
      builder: (context, state, navigationShell) {
        return AppScaffold(navigationShell: navigationShell);
      },
      branches: [
        StatefulShellBranch(
          routes: [
            GoRoute(
              path: '/home',
              builder: (context, state) => const HomePage(),
            ),
          ],
        ),
        StatefulShellBranch(
          preload: true,
          routes: [
            GoRoute(
              path: '/notifications',
              builder: (context, state) => const NotificationsPage(),
            ),
          ],
        ),
      ],
    ),
  ],
);
```

Preloading the branch does not mean the user has navigated to `/notifications`. The active branch, URL, selected tab, and back-stack behavior still belong to `StatefulNavigationShell`. Analytics should therefore record a screen view when the branch becomes active, not when its widget happens to be built during preloading.

## Typed routes use the same decision

With `go_router_builder`, the equivalent option is exposed by `StatefulShellBranchData.$branch`:

```dart
class NotificationsBranch extends StatefulShellBranchData {
  static final $branch = StatefulShellBranchData.$branch(
    preload: true,
  );
}
```

The generated route tree still needs a valid root route for that branch. `preload` controls eager loading of the branch's initial location; it does not make an incomplete branch valid or change the location generated for a typed route.

## The trap: preloading expensive dependencies

I initially treated `preload` as a free performance win. The mistake was putting a repository initialization and a network request directly in the branch page's `initState`. On a cold start, the app now paid for the home screen and notifications before the user had interacted with either one.

Keep the page cheap to construct and make data work cancellable or state-owned:

```dart
class NotificationsPage extends ConsumerWidget {
  const NotificationsPage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final notifications = ref.watch(notificationsProvider);
    return NotificationsView(state: notifications);
  }
}
```

If the provider should fetch only after the tab is visible, leave the branch lazy or gate the fetch on an explicit “tab became active” event. If the data is needed immediately after login, warming the repository outside the route tree is usually clearer than relying on branch construction as a side effect.

## Test the behavior that users can observe

A widget test should distinguish branch construction from branch selection. Navigate to the shell, verify that the eagerly loaded page can build, then select another branch and confirm that its stack still behaves normally. Also test a deep link such as `/notifications/detail/42`: preloading must not replace the requested location with the branch root.

In production, compare cold-start time and first-tab interaction time on the slowest supported device. A smooth second tap is not an improvement if it adds 300 ms to the first frame or triggers duplicate requests.

The practical rule is simple: use `preload: true` for small, high-frequency branch roots with predictable setup. Leave data-heavy or rarely visited branches lazy. `StatefulShellBranch.preload` is a timing control for nested navigation, not a substitute for caching, restoration, or a loading policy.

Reference: [StatefulShellBranch API](https://pub.dev/documentation/go_router/latest/go_router/StatefulShellBranch-class.html) and [Stateful nested navigation](https://pub.dev/documentation/go_router/latest/topics/Configuration-topic.html).
