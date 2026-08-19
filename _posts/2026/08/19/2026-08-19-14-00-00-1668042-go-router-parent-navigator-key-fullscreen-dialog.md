---
layout: post
title: "go_router parentNavigatorKey - Show Flutter Full-Screen Routes Above a Stateful Shell"
description: "Learn how go_router parentNavigatorKey places Flutter login, checkout, and full-screen dialog routes on the root Navigator above persistent tabs."
date: 2026-08-19
tags: [go_router, navigation, Flutter]
comments: true
share: true
---

![go_router root and shell navigators preserving Flutter tab state](/assets/images/go-router-stateful-shell-route.png)

When a Flutter app uses `StatefulShellRoute`, a route can unexpectedly appear inside the current tab instead of covering the whole application. `go_router`'s `parentNavigatorKey` fixes that by choosing which `Navigator` owns the page. The practical result is simple: keep tabs mounted underneath, while showing login, checkout, or a full-screen dialog on the root navigator.

## Why the default navigator is sometimes the wrong one

A stateful shell normally gives every branch its own navigator. That is exactly what we want for tab state: a scroll position in the first tab should survive a trip to the second tab and back.

The same behavior becomes awkward for application-wide pages. Suppose `/cart` is declared inside the shop branch. Without extra configuration, the cart page is pushed onto that branch's navigator. It inherits the tab's navigation context and can look like another tab screen. A modal checkout flow usually needs a different relationship: the current tab remains underneath, but the checkout owns the visible route.

| Route placement | Navigator owner | Typical result |
| --- | --- | --- |
| Default child route | Current shell branch | Page stays inside one tab |
| `parentNavigatorKey: rootKey` | App root navigator | Page covers the shell |
| Separate shell branch | Its own branch navigator | Page behaves like another tab |

`parentNavigatorKey` is therefore a routing decision, not a visual decoration. It changes the stack that receives pushes, pops, back gestures, and restoration state.

## Define a root key and a stateful shell

The root key must be attached to the top-level `GoRouter`. The route that should escape the shell then references that same key.

```dart
final _rootNavigatorKey = GlobalKey<NavigatorState>(debugLabel: 'root');
final _shopNavigatorKey = GlobalKey<NavigatorState>(debugLabel: 'shop');
final _accountNavigatorKey = GlobalKey<NavigatorState>(debugLabel: 'account');

final router = GoRouter(
  navigatorKey: _rootNavigatorKey,
  initialLocation: '/shop',
  routes: [
    StatefulShellRoute.indexedStack(
      builder: (context, state, navigationShell) {
        return AppScaffold(navigationShell: navigationShell);
      },
      branches: [
        StatefulShellBranch(
          navigatorKey: _shopNavigatorKey,
          routes: [
            GoRoute(
              path: '/shop',
              builder: (context, state) => const ShopPage(),
            ),
            GoRoute(
              path: '/shop/product/:id',
              builder: (context, state) {
                return ProductPage(id: state.pathParameters['id']!);
              },
            ),
          ],
        ),
        StatefulShellBranch(
          navigatorKey: _accountNavigatorKey,
          routes: [
            GoRoute(
              path: '/account',
              builder: (context, state) => const AccountPage(),
            ),
          ],
        ),
      ],
    ),
    GoRoute(
      path: '/checkout',
      parentNavigatorKey: _rootNavigatorKey,
      pageBuilder: (context, state) {
        return const MaterialPage(
          fullscreenDialog: true,
          child: CheckoutPage(),
        );
      },
    ),
  ],
);
```

The important part is not `fullscreenDialog`. That flag controls the page transition and platform presentation hints. `parentNavigatorKey` controls where the page is inserted. A normal `MaterialPage` can still be placed on the root navigator, and a full-screen dialog can still be placed on a branch if the keys are configured that way.

## Navigate without losing the active tab

From a product page, call `context.push('/checkout')` when checkout should sit above the current location:

```dart
FilledButton(
  onPressed: () => context.push('/checkout'),
  child: const Text('Buy now'),
)
```

The shell stays mounted because `/checkout` is a child of the top-level route list and targets `_rootNavigatorKey`. When checkout is popped, the user returns to the same product page, with its branch state intact.

There is a subtle distinction between `push` and `go` here. `push` adds a page to the existing navigation stack, which suits a temporary checkout or sign-in flow. `go('/checkout')` changes the declarative location and can replace the current match list. It may still display the root-level page, but it communicates a different state model and can affect what the back button returns to.

## Use a root-level login route for protected flows

The same structure works for authentication. A redirect can send an unauthenticated user to `/login`, while the login page remains independent of whichever tab triggered the action.

```dart
GoRoute(
  path: '/login',
  parentNavigatorKey: _rootNavigatorKey,
  pageBuilder: (context, state) => MaterialPage(
    fullscreenDialog: true,
    child: LoginPage(
      returnLocation: state.uri.queryParameters['from'] ?? '/shop',
    ),
  ),
),
```

Keep the redirect responsible for access policy and `parentNavigatorKey` responsible for presentation hierarchy. Combining both concerns inside a page callback makes deep links harder to reason about, especially when a browser refresh starts directly at `/login`.

## Common mistakes

- Referencing a branch key as the parent for a route that is not inside that branch. The parent key must belong to an ancestor navigator in the route tree.
- Assuming `fullscreenDialog: true` makes a page cover the shell. It does not select the navigator.
- Declaring `/checkout` inside a branch and expecting it to behave like a global flow. Move it to the appropriate top-level route and set the root key explicitly.
- Using `context.go()` for every action. Temporary flows usually need stack semantics, so `push` and `pop` make the return behavior clearer.
- Forgetting that browser URLs still describe the root route. Test a direct refresh at `/checkout`, the browser back button, and Android back navigation.

The useful mental model is to ask one question for every route: “Which navigator should own this page?” Branch-owned pages preserve local tab history. Root-owned pages sit above the shell and can represent global flows. Once that ownership is explicit, `StatefulShellRoute` and full-screen routes work together without manually managing nested `Navigator` stacks.
