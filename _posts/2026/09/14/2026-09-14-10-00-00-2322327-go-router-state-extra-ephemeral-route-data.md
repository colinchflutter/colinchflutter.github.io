---
layout: post
title: "go_router GoRouterState.extra - Pass Ephemeral Data Between Flutter Routes"
description: "Learn how GoRouterState.extra passes temporary Flutter route data, when it disappears, and when URL parameters are the safer choice."
date: 2026-09-14
tags: [go_router, navigation, Flutter, testing, FlutterWeb]
comments: true
share: true
---

![go_router passing typed ephemeral data between Flutter routes](/assets/images/go-router-extra-typed-route-data.png)

`GoRouterState.extra` passes an in-memory object to a Flutter route without adding it to the URL. It fits short-lived workflow data such as a cart preview or confirmation payload. It is the wrong place for an ID that must survive a browser refresh.

## What `extra` actually carries

The value travels with the navigation call. It is not a path parameter, and it does not automatically become JSON.

| Data contract | Good location | What happens after refresh |
| --- | --- | --- |
| Shareable product ID | `/products/:id` | Rebuilt from the URL |
| Search filters | Query parameters | Rebuilt from the URL |
| Temporary draft object | `extra` | May be unavailable |
| Complex temporary object on Web | `extra` + `extraCodec` | Can be restored when encoded |

The last row is a separate design. The [`extraCodec` pattern]({% post_url 2026-08-23-14-00-00-1754457-go-router-extra-codec-browser-safe-route-data %}) addresses browser history serialization. Plain `extra` is process-local state.

## Passing a typed object

Define the smallest object the destination needs, then check its runtime type at the route boundary.

```dart
class CheckoutDraft {
  const CheckoutDraft({required this.productId, required this.quantity});

  final String productId;
  final int quantity;
}

final router = GoRouter(
  routes: [
    GoRoute(
      path: '/checkout',
      builder: (context, state) {
        final draft = state.extra;
        if (draft is! CheckoutDraft) {
          return const MissingDraftPage();
        }
        return CheckoutPage(draft: draft);
      },
    ),
  ],
);
```

Navigate with the same explicit type:

```dart
context.push(
  '/checkout',
  extra: const CheckoutDraft(productId: 'p-42', quantity: 2),
);
```

The `is!` check matters. A deep link, notification, or test can reach `/checkout` without the payload. A direct cast turns that recoverable case into an error screen.

## The failure case on Flutter Web

After `context.push`, the address bar still contains only `/checkout`. Refresh creates a new application instance, so `state.extra` may be missing. The fallback should return to a stable route or reload from a server-side ID.

For data that defines the destination, put the identifier in the route instead:

```dart
GoRoute(
  path: '/checkout/:productId',
  builder: (context, state) {
    final productId = state.pathParameters['productId'];
    if (productId == null || productId.isEmpty) {
      return const MissingProductPage();
    }
    return CheckoutLoader(productId: productId);
  },
)
```

A practical split is simple: the URL owns identity, while `extra` carries optional context. Verify prices and permissions from the server instead of trusting a stale object.

## Testing both navigation paths

Test the missing-payload path as well as the successful push. That is the case a refresh or deep link exposes.

```dart
testWidgets('shows a recovery page when checkout extra is missing', (tester) async {
  final router = GoRouter(initialLocation: '/checkout', routes: [
    GoRoute(
      path: '/checkout',
      builder: (context, state) => state.extra is CheckoutDraft
          ? CheckoutPage(draft: state.extra! as CheckoutDraft)
          : const MissingDraftPage(),
    ),
  ]);

  await tester.pumpWidget(MaterialApp.router(routerConfig: router));
  expect(find.byType(MissingDraftPage), findsOneWidget);
});
```

Keep `extra` small and safe to lose. If losing it makes the route meaningless, promote it to a path/query parameter or define an explicit codec.

### Key points

- `GoRouterState.extra` is in-memory route data, not a shareable URL contract.
- Runtime type checks give deep links and refreshes a controlled fallback.
- Put stable identity in path/query parameters and use `extra` for temporary context.
- Test the missing-payload route, not only the happy-path `push`.
