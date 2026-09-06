---
layout: post
title: "go_router GoRouterState.error - Classify Flutter Navigation Failures Safely"
description: "Learn how GoRouterState.error represents Flutter navigation failures and how to separate user messages, logging, and recovery in go_router."
date: 2026-09-06
tags: [go_router, navigation, testing, Web]
comments: true
share: true
---

![Flutter go_router navigation failure state and recovery screen](https://images.unsplash.com/photo-1551650975-87deedd944c3?w=1200&q=80)

The key detail is the separation between a failed route, its diagnostic data, and the recovery action shown to the user.

The `GoRouterState.error` value should be treated as diagnostic input, not as a ready-made message for a Flutter user. It tells an `errorBuilder` why route matching or page construction failed, while the UI should choose a short recovery message and the logger should keep the original exception and stack trace.

I initially rendered `state.error.toString()` directly on the error page. That was useful during development, but it exposed redirect details and exception text in production. A small classification step makes the boundary safer without hiding useful information from developers.

## What `GoRouterState.error` represents

The error state is produced when `go_router` cannot complete navigation. An unknown path is one example, but a redirect failure or an exception thrown while building a matched route can also arrive here.

| Failure | User-facing result | Developer action |
| --- | --- | --- |
| Unknown deep link | “Page not found” | Record the attempted URI |
| Invalid route data | “This link is no longer valid” | Inspect the exception and parameters |
| Page build exception | “We could not open this page” | Send the error and stack trace to monitoring |

The useful distinction is not the exact exception class. It is whether the user can correct the URL, retry the operation, or should return to a known route.

## Build a safe error boundary

Keep the original error available for logs, but map it to a stable message before rendering. The following router uses `errorBuilder` as the UI boundary and keeps recovery independent from the current navigation stack.

```dart
import 'package:flutter/material.dart';
import 'package:go_router/go_router.dart';

final router = GoRouter(
  routes: [
    GoRoute(
      path: '/',
      builder: (context, state) => const HomePage(),
    ),
  ],
  errorBuilder: (context, state) {
    final error = state.error;
    final message = switch (error) {
      GoException() => 'That page could not be found.',
      FormatException() => 'The link contains invalid data.',
      _ => 'We could not open this page right now.',
    };

    debugPrint('Route failed at ${state.uri}: $error');

    return RouteFailurePage(
      message: message,
      onHome: () => context.go('/'),
    );
  },
);
```

`context.go('/')` is safer than `context.pop()` here. A browser may open the failing URL directly, and a mobile app may have no previous route to pop. The fallback should be a route that exists independently of the failed branch.

The concrete exception types depend on where the failure originated, so the default branch matters. A future package version or an application-specific exception should still produce a useful screen instead of an uncaught switch error.

## Keep the diagnostic path out of the message

`state.uri` is valuable for reproducing the failure, especially when a query parameter or fragment caused the problem. It can contain user-provided data, though, so avoid displaying it as a large exception message. Send it to a privacy-aware logger, redact tokens, and keep the visible copy short.

For tests, exercise both the rendering and recovery contract:

```dart
testWidgets('unknown route offers a safe home action', (tester) async {
  await tester.pumpWidget(MaterialApp.router(routerConfig: router));
  router.go('/missing');
  await tester.pumpAndSettle();

  expect(find.text('That page could not be found.'), findsOneWidget);
  await tester.tap(find.text('Go home'));
  await tester.pumpAndSettle();
  expect(find.byType(HomePage), findsOneWidget);
});
```

`GoRouterState.error` gives the router boundary context, but it does not decide the product response for you. Classify the failure, log the original details, render a stable message, and recover through a known route. That separation keeps deep-link failures understandable in development and calm in production.
