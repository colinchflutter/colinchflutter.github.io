---
layout: post
title: "go_router GoRouterState.pageKey - Preserve Flutter Page State During Route Changes"
description: "Learn how GoRouterState.pageKey controls Flutter page identity, state preservation, and custom transitions in dynamic go_router routes."
date: 2026-09-12
tags: [go_router, navigation, Flutter, performance]
comments: true
share: true
---

![go_router page identity and Flutter navigation state](/assets/images/go-router-stateful-shell-route.png)

`GoRouterState.pageKey` is the value that tells Flutter whether a routed page is the same page as before or a new page. Using it in `MaterialPage` or `CustomTransitionPage` prevents a common Flutter navigation bug: the URL changes, but the old page state survives when it should not, or a form unexpectedly resets during a harmless parameter change.

## The problem: a route URL is not page identity

Imagine a detail route that accepts a product ID:

```dart
GoRoute(
  path: '/products/:productId',
  pageBuilder: (context, state) {
    final productId = state.pathParameters['productId']!;

    return MaterialPage(
      child: ProductPage(productId: productId),
    );
  },
),
```

This renders correctly, but the page has no explicit key. Flutter then has less information when it compares the old and new page configurations. Navigating from `/products/10` to `/products/11` can reuse an element tree in a way that is surprising for local state, focus, animations, or a `TextEditingController` owned by the detail screen.

The useful distinction is small:

| Value | Meaning | Good use |
| --- | --- | --- |
| `state.uri` | The complete concrete URL | Read IDs, query parameters, and fragments |
| `state.path` | The route pattern | Diagnostics and route-level policies |
| `state.pageKey` | Identity of the matched page | Give the Flutter page a stable identity |

`pageKey` is not a replacement for the product ID. It is the routing layer's identity token for the page configuration.

## Pass `pageKey` into `MaterialPage`

The smallest practical change is to forward the key supplied by go_router:

```dart
GoRoute(
  path: '/products/:productId',
  pageBuilder: (context, state) {
    final productId = state.pathParameters['productId'];

    if (productId == null || productId.isEmpty) {
      return const MaterialPage(
        child: ProductNotFoundPage(),
      );
    }

    return MaterialPage(
      key: state.pageKey,
      name: state.name,
      child: ProductPage(productId: productId),
    );
  },
),
```

The `key` belongs to the page, not only to `ProductPage`. This matters because `Page` objects are compared by the Navigator when it builds a new page list. Putting a key on an inner widget may reset that widget, but it does not express the route's identity to Navigator.

In a real app, I initially keyed only the detail widget. That appeared to fix the text field, but the transition stack still behaved inconsistently when the user moved between two product IDs. Moving the key to `MaterialPage` made the ownership clear and fixed both concerns.

## Combine it with a custom transition

`state.pageKey` is also the right key to use when replacing `MaterialPage` with `CustomTransitionPage`:

```dart
GoRoute(
  path: '/products/:productId',
  pageBuilder: (context, state) {
    final productId = state.pathParameters['productId']!;

    return CustomTransitionPage<void>(
      key: state.pageKey,
      name: state.name,
      transitionDuration: const Duration(milliseconds: 240),
      reverseTransitionDuration: const Duration(milliseconds: 180),
      child: ProductPage(productId: productId),
      transitionsBuilder: (context, animation, secondaryAnimation, child) {
        final curved = CurvedAnimation(
          parent: animation,
          curve: Curves.easeOutCubic,
        );

        return FadeTransition(
          opacity: curved,
          child: SlideTransition(
            position: Tween<Offset>(
              begin: const Offset(0.04, 0),
              end: Offset.zero,
            ).animate(curved),
            child: child,
          ),
        );
      },
    );
  },
),
```

The transition code decides how the page moves. `pageKey` decides which page the Navigator is comparing. Mixing those responsibilities is a frequent source of difficult-to-reproduce animation bugs.

## Decide what should reset

The key is most useful when its identity matches the state boundary you want. A product detail page should usually be a new page for a new product ID. A search page should usually keep its scroll position while only its query changes, depending on the UX.

Use a route-specific key only when the default route identity does not match that boundary:

```dart
final productKey = ValueKey<String>('product-$productId');

return MaterialPage(
  key: productKey,
  child: ProductPage(productId: productId),
);
```

I would not replace `state.pageKey` casually. The key from go_router already represents the matched route, and manually constructing one can make nested routes or restoration behavior harder to reason about. A custom key is justified when two distinct state boundaries intentionally share one route definition.

## A test that catches accidental reuse

Give the detail page an observable local value and navigate between two IDs:

```dart
testWidgets('creates a new product page for a new product id', (tester) async {
  final router = GoRouter(
    initialLocation: '/products/10',
    routes: [
      GoRoute(
        path: '/products/:productId',
        pageBuilder: (context, state) => MaterialPage(
          key: state.pageKey,
          child: ProductPage(
            productId: state.pathParameters['productId']!,
          ),
        ),
      ),
    ],
  );

  await tester.pumpWidget(MaterialApp.router(routerConfig: router));
  expect(find.text('Product 10'), findsOneWidget);

  router.go('/products/11');
  await tester.pumpAndSettle();

  expect(find.text('Product 11'), findsOneWidget);
});
```

The test does not assert the internal key directly. It verifies the user-visible boundary that the key is meant to protect. Add a stateful child, focus behavior, or transition callback when the bug involves more than the page title.

## Practical rules

- Put `state.pageKey` on the `Page` returned by `pageBuilder`.
- Use `pathParameters` and `uri` for values; do not parse `pageKey` as a URL.
- Keep `name` separate from the key so logs and restoration remain readable.
- Use a custom `ValueKey` only when the route's default identity is not your desired state boundary.
- Test navigation between two concrete parameter values, not only the initial location.

`GoRouterState.pageKey` looks like a small property, but it connects go_router's route matching to Flutter Navigator's page diffing. Once the key is attached at the page boundary, dynamic detail routes, local form state, and custom transitions become much easier to predict.
