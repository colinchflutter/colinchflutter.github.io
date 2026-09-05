---
layout: post
title: "go_router GoRouterState.pathParameters - Parse Flutter Route IDs Safely"
description: "Learn how go_router GoRouterState.pathParameters carries Flutter route IDs and how to validate missing or malformed values in deep links."
date: 2026-09-05
tags: [go_router, navigation, testing, Web]
comments: true
share: true
---
![go_router GoRouterState.pathParameters for Flutter deep links](/assets/images/go-router-stateful-shell-route.png)

`GoRouterState.pathParameters` is the right place to read dynamic values such as `orderId` or `userId` from a Flutter route. It keeps route parsing close to the route definition and avoids splitting URL strings manually. The important detail is that the map contains strings, so the page still needs an explicit validation step before it calls a repository or builds a detail screen.

## The problem with treating every path value as valid

Consider a route like `/orders/:orderId`. A button inside the app may only navigate with numeric IDs, but a browser can open `/orders/abc`, `/orders/`, or a bookmarked URL for an order that no longer exists. A force unwrap hides that distinction:

```dart
final orderId = state.pathParameters['orderId']!;
return OrderPage(orderId: int.parse(orderId));
```

This can fail in two different ways. A missing key throws before the page is built, while a non-numeric value makes `int.parse` throw. Neither failure gives the user a useful route-level response.

| Value | Meaning | Recommended action |
|---|---|---|
| `state.pathParameters['orderId']` | The raw route parameter string | Read at the route boundary |
| `state.uri.queryParameters['tab']` | Optional query state | Do not mix with path identity |
| `state.matchedLocation` | Concrete matched path | Use for route-aware display |
| `state.fullPath` | Route template | Use for grouping and diagnostics |

## Parse the parameter at the route boundary

The route builder is a good place to turn an untrusted string into the type the page actually needs. Returning a small error widget keeps malformed deep links local and makes the page constructor honest.

```dart
final router = GoRouter(
  routes: [
    GoRoute(
      path: '/orders/:orderId',
      builder: (context, state) {
        final orderId = int.tryParse(
          state.pathParameters['orderId'] ?? '',
        );

        if (orderId == null || orderId <= 0) {
          return const InvalidOrderLinkPage();
        }

        return OrderPage(orderId: orderId);
      },
    ),
  ],
);
```

The page now receives an `int`, not a value that may or may not be usable. That boundary also handles a direct browser request and an in-app `context.go('/orders/42')` consistently.

For several routes, I prefer a small helper rather than repeating slightly different validation rules:

```dart
int? positiveId(GoRouterState state, String key) {
  final raw = state.pathParameters[key];
  final value = raw == null ? null : int.tryParse(raw);
  return value != null && value > 0 ? value : null;
}

GoRoute(
  path: '/orders/:orderId',
  builder: (context, state) {
    final id = positiveId(state, 'orderId');
    return id == null
        ? const InvalidOrderLinkPage()
        : OrderPage(orderId: id);
  },
),
```

Keep the helper deliberately narrow. A route parameter parser should validate syntax and basic shape; it should not fetch the order or decide whether the current account may view it. Those are repository and authorization concerns.

## Path parameters versus query parameters

A path parameter identifies the resource. A query parameter usually changes how that resource is displayed. For `/orders/42?tab=items`, `42` belongs in `pathParameters`, while `items` belongs in `state.uri.queryParameters`:

```dart
final orderId = positiveId(state, 'orderId');
final tab = state.uri.queryParameters['tab'] ?? 'overview';

if (orderId == null) {
  return const InvalidOrderLinkPage();
}

return OrderPage(orderId: orderId, initialTab: OrderTab.parse(tab));
```

Do not use `matchedLocation` to recover the ID by splitting on `/`. That couples the code to the current URL shape and becomes fragile when a parent route or prefix changes. `pathParameters` already expresses the router's interpretation of the path.

## Test the cases that users actually open

The useful tests are not limited to the happy-path navigation button. Include a valid ID, a missing value, and a malformed value. The exact widget assertions depend on the app, but the contract should look like this:

```dart
testWidgets('rejects a malformed order deep link', (tester) async {
  final router = GoRouter(
    initialLocation: '/orders/not-a-number',
    routes: [orderRoute],
  );

  await tester.pumpWidget(
    MaterialApp.router(routerConfig: router),
  );
  await tester.pumpAndSettle();

  expect(find.byType(InvalidOrderLinkPage), findsOneWidget);
  expect(find.byType(OrderPage), findsNothing);
});
```

One trap is validating only inside a data-loading widget. By then, the invalid string may already have reached a repository, a cache key, or an analytics event. Parse once at the route boundary, then pass the typed value downward.

The practical split is simple: use `pathParameters` for resource identity, `queryParameters` for optional view state, and `fullPath` or `matchedLocation` for route metadata. That separation makes `go_router` deep links safer without turning every page into a URL parser.
