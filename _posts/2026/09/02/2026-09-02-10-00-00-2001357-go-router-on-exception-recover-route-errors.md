---
layout: post
title: "go_router onException - Recover from Flutter Route Errors Before the Error Page"
description: "Learn how go_router onException handles Flutter deep-link and parsing failures, redirects safely, and differs from errorBuilder."
date: 2026-09-02
tags: [go_router, navigation, error_handling, Flutter, Web]
comments: true
share: true
---

![go_router handling a Flutter route exception and recovering to a safe page](/assets/images/go-router-observers-navigation-scope.png)

The safest place to recover from an invalid Flutter deep link is `GoRouter.onException`. It receives the failed `GoRouterState` and the router itself, so the app can send a malformed URL to a safe route instead of exposing a generic error screen. The important detail is that `onException` is a recovery callback, while `errorBuilder` is a rendering callback.

## Why `errorBuilder` is not always enough

An unknown path, a malformed path parameter, or a redirect loop can all end in an error state. `errorBuilder` can render that state, but it does not own the navigation decision. If the desired behavior is “show the catalog and keep the app usable,” handling the exception at the router boundary is clearer.

| API | Main responsibility | Typical result |
| --- | --- | --- |
| `onException` | Decide how to recover | Redirect or push a safe page |
| `errorPageBuilder` | Build a `Page` for an error | Custom transition or page identity |
| `errorBuilder` | Build a widget for an error | Inline error screen |

The current `go_router` API uses this callback shape:

```dart
typedef GoExceptionHandler = void Function(
  BuildContext context,
  GoRouterState state,
  GoRouter router,
);
```

## Redirect an invalid deep link

Use the callback on `GoRouter`. The failed exception is available through `state.error`; the original URI is still useful for logging or deciding whether the failure came from a public link.

```dart
final router = GoRouter(
  initialLocation: '/catalog',
  routes: [
    GoRoute(
      path: '/catalog',
      builder: (context, state) => const CatalogPage(),
    ),
    GoRoute(
      path: '/products/:productId',
      builder: (context, state) {
        return ProductPage(id: state.pathParameters['productId']!);
      },
    ),
  ],
  onException: (context, state, router) {
    debugPrint(
      'Navigation failed at ${state.uri}: ${state.error}',
    );

    router.go('/catalog');
  },
);
```

This handles a browser opening `/products/not-a-real-route` when the route configuration cannot produce a match. `router.go` starts a new location; it does not attempt to pop the failed route. That makes it a better fit for a broken initial deep link than `context.pop()`, because there may be no previous page to return to.

## Do not create an exception loop

The recovery target must be outside the failure condition. A common mistake is redirecting every exception to a route that itself requires missing data:

```dart
onException: (context, state, router) {
  router.go('/products/${state.pathParameters['productId']}');
},
```

If parsing `productId` caused the original failure, this callback simply requests the same invalid location again. Pick a static route such as `/catalog`, and keep that route independent from authentication or product lookup while recovering.

For an authenticated app, the fallback can be a public error or sign-in route. Avoid calling a redirecting route blindly if its own guard can fail for the same reason:

```dart
onException: (context, state, router) {
  final isPublicPath = state.uri.path == '/catalog';
  if (!isPublicPath) {
    router.go('/catalog');
  }
},
```

The guard is small, but it prevents a recovery callback from becoming another source of repeated navigation events.

## `onException` versus `errorBuilder`

`onException` takes precedence when it is configured. If it is absent, `go_router` can pass the failure to `errorPageBuilder`, then `errorBuilder`. This means adding both does not produce two screens for one failure. The exception handler gets the first decision.

I use `errorBuilder` when the error itself is the intended product experience—for example, a branded 404 page with a “Back to home” button. I use `onException` when the app should recover silently or route the user to a known safe destination. A hybrid setup is also useful: handle expected malformed links in `onException`, and keep `errorBuilder` for failures that deserve an explanation.

## Test the browser case separately

Button navigation alone will not cover deep-link failures. Add tests for both a normal route and a URL that cannot be matched:

```dart
testWidgets('recovers from an invalid product link', (tester) async {
  final router = GoRouter(
    initialLocation: '/products/missing',
    routes: [
      GoRoute(
        path: '/catalog',
        builder: (context, state) => const Text('Catalog'),
      ),
      GoRoute(
        path: '/products/:id',
        builder: (context, state) => const Text('Product'),
      ),
    ],
    onException: (context, state, router) => router.go('/catalog'),
  );

  await tester.pumpWidget(
    MaterialApp.router(routerConfig: router),
  );
  await tester.pumpAndSettle();

  expect(find.text('Catalog'), findsOneWidget);
});
```

Also test a browser refresh on Flutter Web, a malformed path parameter, and the fallback route itself. A route can work after a button tap but fail on startup because the initial location has no previous route to restore.

## Practical rules

- Use `onException` when the app must choose a recovery location.
- Use `errorBuilder` or `errorPageBuilder` when the error should be rendered.
- Keep the fallback route static and independently buildable.
- Log `state.uri` and `state.error`, but avoid exposing sensitive query data in production logs.
- Test initial deep links and browser refreshes, not only `context.go()` from a widget.

`go_router.onException` turns route parsing failures into an explicit navigation policy. Once the fallback route cannot fail for the same input, invalid Flutter URLs become recoverable application states instead of dead ends.

References: [go_router error handling](https://pub.dev/documentation/go_router/latest/topics/Error%20handling-topic.html) and [GoExceptionHandler API](https://pub.dev/documentation/go_router/latest/go_router/GoExceptionHandler.html).
