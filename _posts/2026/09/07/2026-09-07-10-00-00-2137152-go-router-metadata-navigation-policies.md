---
layout: post
title: "go_router metadata - Build Route-Aware Flutter Navigation Policies"
description: "Learn how go_router metadata and GoRouterState.metadata add typed route hints for Flutter analytics, accessibility, and navigation policies."
date: 2026-09-07
tags: [go_router, navigation, Flutter, accessibility, testing]
comments: true
share: true
---

![go_router route metadata controlling Flutter navigation policies](/assets/images/go-router-stateful-shell-route.png)

`go_router` metadata gives a Flutter route an explicit set of application-defined hints. That is useful when a policy depends on what a route means, not on whether its URL happens to start with `/admin` or `/checkout`. With `GoRoute.metadata` and `GoRouterState.metadata`, I can keep analytics labels, accessibility behavior, and navigation rules close to the route declaration while keeping widgets free of string-based route checks.

## Why URL checks become a maintenance problem

This kind of condition looks harmless:

```dart
final isCheckout = state.uri.path.startsWith('/checkout');
```

It becomes fragile when the route is renamed, nested under a shell, or split into `/checkout/review` and `/checkout/payment`. The policy is encoded in a URL convention that every caller has to remember. A route name is better for identity, but it still does not describe product-level properties such as “requires an unsaved-change warning” or “hide the bottom navigation bar.”

Route metadata is a small map for those properties. The route tree remains the source of truth, and the policy can read the metadata exposed on the current `GoRouterState`.

| Route information | Best use | Example |
| --- | --- | --- |
| `state.uri` | Exact deep link | `/orders/42?tab=history` |
| `state.fullPath` | Stable route template | `/orders/:orderId` |
| `state.metadata` | Application-defined policy | `requiresAuth: true` |
| `state.pathParameters` | Resource identity | `orderId: 42` |

## Declare metadata on the route

The `metadata` argument belongs on `GoRoute`. Keep values small, immutable, and easy to validate. I prefer constants for keys so a typo does not silently disable a policy.

```dart
import 'package:flutter/material.dart';
import 'package:go_router/go_router.dart';

const requiresAuthKey = 'requiresAuth';
const sectionKey = 'section';
const hideShellKey = 'hideShell';

final router = GoRouter(
  routes: [
    GoRoute(
      path: '/orders',
      name: 'orders',
      metadata: const {
        sectionKey: 'orders',
        requiresAuthKey: true,
      },
      builder: (context, state) => const OrdersPage(),
      routes: [
        GoRoute(
          path: ':orderId',
          metadata: const {hideShellKey: true},
          builder: (context, state) => OrderPage(
            id: state.pathParameters['orderId']!,
          ),
        ),
      ],
    ),
  ],
);
```

The child route overrides only `hideShell`; it does not need to repeat the parent’s section or authentication metadata. This is the useful difference from passing a separate configuration object into every page.

## Read inherited metadata in a redirect

`GoRouterState.metadata` represents the metadata accumulated for the matched route. A parent value can be inherited, while a child can override the same key. That makes route-level guards readable:

```dart
String? appRedirect(BuildContext context, GoRouterState state) {
  final requiresAuth = state.metadata[requiresAuthKey] == true;
  final isSignIn = state.matchedLocation == '/sign-in';
  final signedIn = SessionScope.of(context).isSignedIn;

  if (requiresAuth && !signedIn && !isSignIn) {
    return GoRouter.of(context).namedLocation(
      'sign-in',
      queryParameters: {'from': state.uri.toString()},
    );
  }
  return null;
}
```

Register the function as the router-level redirect when the rule applies across the tree:

```dart
final router = GoRouter(
  redirect: appRedirect,
  routes: appRoutes,
);
```

The important detail is the boolean comparison. `state.metadata[requiresAuthKey] == true` treats a missing value, `null`, and an accidentally malformed value as “not protected.” If the application must fail closed, use a typed reader that throws during development and reports the bad route configuration in tests.

## Use metadata for shell UI without parsing paths

A persistent shell often needs to decide whether to show a bottom bar or a compact app bar. Passing `GoRouterState` into the shell builder gives it the same route metadata:

```dart
ShellRoute(
  builder: (context, state, child) {
    final hideShell = state.metadata[hideShellKey] == true;

    return Scaffold(
      body: child,
      bottomNavigationBar: hideShell
          ? null
          : const NavigationBar(destinations: [
              NavigationDestination(
                icon: Icon(Icons.list),
                label: 'Orders',
              ),
            ]),
    );
  },
  routes: appRoutes,
)
```

There is a trap here: with `StatefulShellRoute`, each branch has its own navigator and the shell may remain mounted while the active child changes. Test the actual branch structure, not only a flat `GoRoute` list. If metadata is declared on a child route, inspect the state delivered at the shell boundary and move truly shell-wide metadata to the shell’s route configuration when appropriate.

## Keep analytics and policy metadata separate

It is tempting to put every screen label and event name in one large map. That creates an untyped mini-database inside the route tree. I keep metadata limited to stable behavior and derive dynamic values from route state:

```dart
void trackRoute(GoRouterState state) {
  final section = state.metadata[sectionKey] as String?;

  analytics.track('screen_view', {
    'screen': state.fullPath ?? state.matchedLocation,
    'section': section,
    'order_id': state.pathParameters['orderId'],
  });
}
```

Do not store user names, tokens, or large model objects in metadata. Metadata describes the route configuration; it is not a replacement for `extra`, a state manager, or a server response. Keep shareable state in path and query parameters, and keep secrets out of all navigation state.

## Tests that catch the real failures

The most valuable test is not “the map contains a key.” It is a navigation test that proves the policy follows a nested route:

```dart
testWidgets('order details inherits auth metadata', (tester) async {
  final router = buildRouter(session: const SignedOutSession());
  await tester.pumpWidget(TestApp(router: router));

  router.go('/orders/42');
  await tester.pumpAndSettle();

  expect(router.location, '/sign-in?from=%2Forders%2F42');
});
```

Also test a child override, a missing metadata key, a browser refresh/deep link, and a `StatefulShellRoute` branch switch. Those cases exposed the mistakes in my first implementation: I checked metadata only in page builders, duplicated the auth flag on every child, and assumed a shell would rebuild at the same boundary as the leaf page.

The practical rule is simple: use URL fields for navigation identity, and use `metadata` for stable route policy. A few deliberate keys can remove a surprising amount of duplicated path parsing, while keeping redirects and shell behavior aligned with the route tree.
