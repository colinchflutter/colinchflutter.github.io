---
layout: post
title: "go_router StatefulShellRoute - Preserve Flutter Tab State Across Deep Links"
description: "Use go_router StatefulShellRoute to build Flutter tab navigation that preserves each branch state while handling nested routes and deep links cleanly."
date: 2026-08-16
tags: [go_router, navigation, state_management, Flutter]
comments: true
share: true
---

![go_router StatefulShellRoute preserving Flutter tab navigation while a nested detail route opens](/assets/images/go-router-stateful-shell-route.png)

*The key detail is that each tab owns a navigation branch, so switching tabs does not throw away the branch's scroll position or nested route.*

`go_router`'s `StatefulShellRoute` is the right tool when a Flutter app has bottom tabs and each tab needs its own navigation history. A plain `ShellRoute` can share a layout, but it does not automatically preserve a separate stack for every branch. That difference becomes obvious when a user opens a detail page, switches tabs, and comes back expecting the same screen and scroll position.

## Why a normal ShellRoute can feel wrong

I initially placed a `BottomNavigationBar` inside one `ShellRoute` and changed the location with `context.go('/feed')`. The URLs changed correctly, but the tab stacks behaved like one shared history. A nested detail page disappeared when the user moved to another tab, and rebuilding the branch also reset its list state.

`StatefulShellRoute.indexedStack` changes the ownership model:

| Piece | Responsibility |
| --- | --- |
| Stateful shell | Keeps the tab layout mounted |
| Branch | Owns one tab's navigator and route stack |
| `goBranch` | Switches branches without rebuilding their stacks |
| `initialLocation` | Decides whether a tab switch returns to its root |

The shell is not a global replacement for navigation. It is a container for several independent navigators.

## Define one branch per tab

The following example has home, search, and account branches. The detail route is nested under the home branch, which means `/home/article/42` is still part of that branch's history.

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
              routes: [
                GoRoute(
                  path: 'article/:id',
                  builder: (context, state) => ArticlePage(
                    id: state.pathParameters['id']!,
                  ),
                ),
              ],
            ),
          ],
        ),
        StatefulShellBranch(
          routes: [
            GoRoute(
              path: '/search',
              builder: (context, state) => const SearchPage(),
            ),
          ],
        ),
        StatefulShellBranch(
          routes: [
            GoRoute(
              path: '/account',
              builder: (context, state) => const AccountPage(),
            ),
          ],
        ),
      ],
    ),
  ],
);
```

Notice that the child path is `article/:id`, not `/article/:id`. A leading slash makes it an absolute path and breaks the intended parent-child relationship. This is a small typo with a surprisingly confusing result: the page may still appear, but its URL no longer reflects the branch hierarchy you designed.

## Connect the shell to the bottom navigation bar

`StatefulNavigationShell` provides the branch index and the method that activates another branch. Keep the tab UI in the shell builder so it remains mounted while branch navigators change.

```dart
class AppScaffold extends StatelessWidget {
  const AppScaffold({required this.navigationShell, super.key});

  final StatefulNavigationShell navigationShell;

  void _onTap(int index) {
    navigationShell.goBranch(
      index,
      // Tap the active tab again to return to its root.
      initialLocation: index == navigationShell.currentIndex,
    );
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: navigationShell,
      bottomNavigationBar: NavigationBar(
        selectedIndex: navigationShell.currentIndex,
        onDestinationSelected: _onTap,
        destinations: const [
          NavigationDestination(icon: Icon(Icons.home), label: 'Home'),
          NavigationDestination(icon: Icon(Icons.search), label: 'Search'),
          NavigationDestination(icon: Icon(Icons.person), label: 'Account'),
        ],
      ),
    );
  }
}
```

The `initialLocation` choice is easy to get backwards. Setting it to `true` for every tap makes each tab jump to its root, which defeats the point of independent stacks. Setting it to `true` only when the current tab is tapped gives the familiar “tap active tab to reset” behavior while a normal tab switch restores the previous nested page.

## Deep links and branch state

If the app starts at `/home/article/42`, `go_router` selects the home branch and builds the article route directly. There is no need to manually open the home tab first. After the user visits search and returns to home, the article route remains the active location because the home navigator owns it.

There are two practical boundaries to keep in mind:

- A branch preserves navigator state, not arbitrary data. Reloading a Flutter Web page still requires restoring data from a repository or URL.
- A detail page that belongs to every tab should usually be declared outside the shell. Duplicating it in three branches creates three independent route identities and complicates deep-link handling.

For a global detail page, put the route beside the shell and navigate with a full location. For a tab-specific detail page, keep it nested under the relevant `StatefulShellBranch`.

## The failure cases I check first

When a stateful shell behaves unexpectedly, I check the route tree before changing widget state. The most common mistakes are using `context.go` for a tab index without matching branch paths, putting a nested route at the wrong level, and recreating the `GoRouter` inside `build`. The router should normally be long-lived; rebuilding it can recreate navigators and erase the state the shell was meant to preserve.

The useful mental model is simple: a `ShellRoute` gives you a shared frame, while a `StatefulShellRoute` gives you a shared frame plus independent branch navigators. Once each tab owns the routes that belong to it, deep links, back navigation, and tab restoration become predictable instead of a collection of special cases.
