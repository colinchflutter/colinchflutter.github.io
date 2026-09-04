---
layout: post
title: "go_router GoRouterState.name - Build Stable Flutter Route Policies"
description: "Learn how GoRouterState.name identifies Flutter routes for breadcrumbs, access policies, and analytics without coupling code to URL strings."
date: 2026-09-05
tags: [go_router, navigation, Flutter, testing]
comments: true
share: true
---

![go_router route names organizing a Flutter navigation tree](/assets/images/go-router-stateful-shell-route.png)

`GoRouterState.name` gives a Flutter app the logical name of the route that produced the current state. It is useful when a breadcrumb, page policy, or analytics event needs a stable identifier, while the visible URL may contain changing IDs and query parameters. The value is nullable, so production code should treat it as metadata rather than an unchecked replacement for `state.uri`.

## Why a route name is different from a URL

I used to derive a section label from `state.uri.path.startsWith('/orders')`. That worked until a nested route and an alias were added. An order detail page carried an ID, and a renamed URL changed the UI policy even though the product concept had not changed.

These values answer different questions:

| Value | Describes | Suitable for |
| --- | --- | --- |
| `state.uri` | The concrete URL, including query data | Deep links and filters |
| `state.matchedLocation` | The matched location string | URL-oriented logging |
| `state.fullPath` | The route template | Grouping parameterized screens |
| `state.name` | The configured logical route name | Policies, labels, and event keys |

Give names to routes that are part of an application contract. A generated or unnamed route can legitimately produce `null`, especially when a route is assembled dynamically.

## Define names at the route boundary

The route name should describe the destination, not the widget class or the current record. Keep the ID in `pathParameters` and keep the route name stable when the URL changes.

```dart
final router = GoRouter(
  routes: [
    GoRoute(
      path: '/orders',
      name: 'orders',
      builder: (context, state) => const OrdersPage(),
      routes: [
        GoRoute(
          path: ':orderId',
          name: 'orderDetails',
          builder: (context, state) => OrderDetailsPage(
            orderId: state.pathParameters['orderId']!,
          ),
        ),
      ],
    ),
  ],
);
```

The detail screen still has the logical name `orderDetails` for `/orders/42` and `/orders/43`. That makes a route policy independent from the resource value.

## Use `name` for route-aware UI

Read the state inside the page or a route-aware shell. A small mapping keeps presentation labels out of URL parsing code.

```dart
String sectionLabel(GoRouterState state) {
  switch (state.name) {
    case 'orders':
    case 'orderDetails':
      return 'Orders';
    case 'settings':
      return 'Settings';
    default:
      return 'Home';
  }
}
```

This is a good fit for a shared app bar, but not for deciding whether a page is currently visible in a `StatefulShellRoute`. A shell can keep several branch navigators alive. Use the shell's selected index or the active match information for visibility, and use `name` for the semantic identity of the matched destination.

## Centralize access policies

Names also make a compact policy table possible. The policy function should return a redirect location, not perform navigation itself.

```dart
String? policyRedirect(GoRouterState state, AuthState auth) {
  const protectedRoutes = {'billing', 'teamSettings'};

  if (protectedRoutes.contains(state.name) && !auth.isSignedIn) {
    return '/sign-in?from=${Uri.encodeComponent(state.uri.toString())}';
  }
  return null;
}
```

For a nested page, verify which `GoRoute` name is exposed by the state in the version of `go_router` used by the project. A policy that expects `settings` can fail silently if the actual state name is `teamSettings`. I keep route names in constants or generated typed-route code when many guards depend on them.

## Testing the contract

Names are configuration, so a routing test should catch accidental renames. Test the router's generated location separately from the policy's logical key.

```dart
test('order detail route name keeps its URL contract', () {
  expect(
    router.namedLocation(
      'orderDetails',
      pathParameters: {'orderId': '42'},
    ),
    '/orders/42',
  );
});
```

An integration-style navigation test can also navigate to `/orders/42` and inspect the rendered policy output. Repeat that flow after a browser refresh if the app targets Flutter Web; the URL contract and the logical route name should remain aligned.

## Practical boundaries

Do not use `name` as a URL. It is not a deep link, and it does not carry path or query parameters. Do not assume it is non-null, and do not use it as a substitute for `pageKey` when the question is Flutter page identity or state preservation. For analytics, `name` is a useful event key, while `fullPath` and `uri` can remain separate diagnostic fields.

The useful split is straightforward: `uri` describes what the user opened, `fullPath` describes the route template, and `name` describes the application-level destination. Keeping those contracts separate makes `go_router` policies easier to test and keeps route-aware Flutter UI stable when URLs evolve.
