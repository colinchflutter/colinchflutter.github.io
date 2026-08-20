---
layout: post
title: "go_router Path Parameters - Parse Flutter Route IDs Safely"
description: "Learn how to validate go_router pathParameters in Flutter before turning URL segments into IDs, with a safe redirect and typed route example."
date: 2026-08-20
tags: [go_router, navigation, testing, performance]
comments: true
share: true
---

![go_router path parameters carrying a validated Flutter route ID](/assets/images/go-router-stateful-shell-route.png)

`go_router` path parameters are always received as strings. The safe Flutter pattern is to keep the URL readable, validate the segment at the router boundary, and only then build a screen with an `int` or domain ID. Passing an invalid value deeper into the widget tree makes the eventual error harder to understand.

## The boundary between URL and app data

A route such as `/orders/:orderId` can receive `/orders/42` or `/orders/not-a-number`. Both match the route pattern. Matching only checks the shape of the path; it does not prove that `orderId` is usable by the app.

| URL segment | `pathParameters` value | App decision |
|---|---|---|
| `/orders/42` | `"42"` | Parse and load order 42 |
| `/orders/not-a-number` | `"not-a-number"` | Redirect or show a route error |
| Missing segment | No match | Let `errorBuilder` handle it |

The important distinction is that a path parameter identifies a resource, while a query parameter usually changes a view such as filtering or sorting. Keeping that distinction makes deep links easier to test and share.

## Validate before building the page

The redirect is a good place for structural validation. It runs before the page is constructed, so malformed links do not create a screen with an impossible ID.

```dart
final router = GoRouter(
  routes: [
    GoRoute(
      name: 'order',
      path: '/orders/:orderId',
      redirect: (context, state) {
        final rawId = state.pathParameters['orderId'];
        final orderId = int.tryParse(rawId ?? '');

        if (orderId == null || orderId <= 0) {
          return '/not-found';
        }
        return null;
      },
      builder: (context, state) {
        final orderId = int.parse(state.pathParameters['orderId']!);
        return OrderPage(orderId: orderId);
      },
    ),
    GoRoute(
      path: '/not-found',
      builder: (context, state) => const NotFoundPage(),
    ),
  ],
);
```

The second parse is safe because the redirect already rejected invalid values. If the validation rule becomes more complicated, extract it into one function so the redirect and builder cannot drift apart. For UUIDs or slugs, replace `int.tryParse` with a small value-object parser rather than accepting any non-empty string.

## Traps that appear in production

Do not read a path parameter once in `initState` and assume it never changes. A reused page can receive a new route state; use the route value as the source of truth or give the page a key based on the ID. Also avoid putting a required resource ID in `extra`: `extra` is not reliably preserved across browser refreshes or an externally opened deep link.

Test both the valid and malformed URLs. A small routing matrix catches more than a happy-path widget test:

```dart
expect(router.namedLocation('order', pathParameters: {'orderId': '42'}), '/orders/42');
// Navigate to /orders/0 and /orders/nope, then expect NotFoundPage.
```

The rule is simple: let `go_router` match the URL, validate its strings at the boundary, and pass typed data to the page only after that check succeeds.
