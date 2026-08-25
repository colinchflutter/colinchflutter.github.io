---
layout: post
title: "go_router pageKey - Build Stable Custom Pages and Transitions in Flutter"
description: "Learn how go_router pageKey identifies Flutter routes and how to use it with pageBuilder for stable custom pages and transitions."
date: 2026-08-25
tags: [go_router, navigation, Flutter, animation, performance]
comments: true
share: true
---

![go_router custom pages and route identity in Flutter](/assets/images/go-router-stateful-shell-route.png)

`go_router` gives each matched route a `pageKey`, and that key is more than boilerplate for a `MaterialPage`. It tells Flutter whether a page is the same route as before or a new page that should be inserted, removed, or animated. If a custom `pageBuilder` behaves strangely, the page key is one of the first things to inspect.

## Why the key matters

`builder` is convenient because `go_router` wraps the returned widget in a page for you. A custom transition requires `pageBuilder`, which returns a `Page` directly. The safest default is to pass `state.pageKey` to that page:

```dart
GoRoute(
  path: '/details/:id',
  pageBuilder: (context, state) {
    final id = state.pathParameters['id']!;

    return CustomTransitionPage<void>(
      key: state.pageKey,
      child: DetailsScreen(id: id),
      transitionsBuilder: (context, animation, secondaryAnimation, child) {
        return FadeTransition(opacity: animation, child: child);
      },
    );
  },
),
```

The page key is derived from the matched route. For `/details/10` and `/details/11`, the route is the same template but the page identity can change with the matched location. That gives Flutter enough information to compare the old and new page during navigation.

| Page configuration | Typical result |
| --- | --- |
| `key: state.pageKey` | Route changes can produce a new page and transition |
| No key on a custom page | Flutter may not compare route instances as intended |
| A constant key for every route | Different destinations can be treated as the same page |
| A manually generated key on every build | Unnecessary page replacement and state loss |

The practical rule is simple: use `state.pageKey` unless you have a specific reason to define another identity.

## A reusable slide page

When several routes need the same transition, keep the page type in one place. This prevents small differences in `reverseTransitionDuration`, curves, or key handling from spreading through the router:

```dart
class SlidePage<T> extends CustomTransitionPage<T> {
  SlidePage({
    required LocalKey key,
    required Widget child,
  }) : super(
          key: key,
          child: child,
          transitionDuration: const Duration(milliseconds: 260),
          reverseTransitionDuration: const Duration(milliseconds: 220),
          transitionsBuilder:
              (context, animation, secondaryAnimation, child) {
            final offset = Tween<Offset>(
              begin: const Offset(1, 0),
              end: Offset.zero,
            ).chain(CurveTween(curve: Curves.easeOutCubic));

            return SlideTransition(
              position: animation.drive(offset),
              child: child,
            );
          },
        );
}

GoRoute(
  path: '/settings',
  pageBuilder: (context, state) => SlidePage<void>(
    key: state.pageKey,
    child: const SettingsScreen(),
  ),
),
```

The route configuration owns the identity, while `SlidePage` owns the visual behavior. That separation becomes useful when the same screen is reached from a deep link, a bottom navigation branch, or a button inside another page.

## `builder` versus `pageBuilder`

Use `builder` when the default Material or Cupertino page behavior is enough. Use `pageBuilder` when the route needs a custom transition, a fullscreen dialog, a custom restoration ID, or another `Page`-level setting.

```dart
GoRoute(
  path: '/compose',
  pageBuilder: (context, state) => MaterialPage<void>(
    key: state.pageKey,
    fullscreenDialog: true,
    child: const ComposeScreen(),
  ),
),
```

A common mistake is to return `MaterialPage` without a key because the screen still appears to work. The problem usually shows up later: a form keeps the wrong state, a transition does not run, or a page is rebuilt when only its parameters changed. The missing key is not always visible in a short manual test.

## Do not create a new identity on every build

Avoid `UniqueKey()` in a route's `pageBuilder` unless throwing away the current page state is intentional. Redirect refreshes, inherited state changes, or query updates can cause the builder to run again. A fresh key tells Flutter that the old page is unrelated, so text controllers, scroll positions, and animations may be discarded.

`state.pageKey` is stable for the route match while still changing when the matched page identity changes. That is usually the behavior a URL-based application wants.

## Testing the contract

For a custom page, test both navigation and identity-sensitive behavior. A small widget test can verify that the destination is present after navigation:

```dart
testWidgets('opens settings with the routed page', (tester) async {
  await tester.pumpWidget(
    MaterialApp.router(routerConfig: router),
  );

  router.go('/settings');
  await tester.pumpAndSettle();

  expect(find.byType(SettingsScreen), findsOneWidget);
});
```

For a form or scroll-heavy screen, add an assertion that state survives an unrelated router refresh. Also test a parameter change such as `/details/10` to `/details/11`; that is where an accidental constant key or `UniqueKey` tends to reveal itself.

`pageKey` is a small part of the `go_router` API, but it connects URL matching, Flutter page identity, state preservation, and transitions. Start with `state.pageKey`, customize the `Page` only where needed, and make any different identity rule an explicit design decision.

References: [GoRouterState API](https://pub.dev/documentation/go_router/latest/go_router/GoRouterState-class.html) and [RouteBase API](https://pub.dev/documentation/go_router/latest/go_router/RouteBase-class.html).
