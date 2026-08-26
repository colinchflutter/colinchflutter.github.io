---
layout: post
title: "go_router onException - Recover Flutter Route Errors Without a Blank Screen"
description: "Learn how go_router onException handles Flutter route failures, preserves useful diagnostics, and redirects users to a safe recovery page."
date: 2026-08-27
tags: [go_router, navigation, error_handling, Web]
comments: true
share: true
---

![go_router onException recovering from a failed Flutter route](/assets/images/go-router-extra-typed-route-data.png)

`go_router`'s `onException` callback is the boundary for navigation failures that happen before a route can render. It is useful for malformed deep links, blocked initial navigation, and route parsing errors that would otherwise leave the user on an error page or an unusable screen. The important detail is that `onException` handles a failure; it does not replace normal route authorization or form-exit checks.

## Separate navigation errors from navigation policy

Several callbacks look similar, but they answer different questions:

| Callback | Question it answers | Typical use |
| --- | --- | --- |
| `redirect` | Should this location become another location? | Authentication |
| `onEnter` | May the destination be entered? | Permission or preflight checks |
| `onExit` | May the current route be removed? | Unsaved form confirmation |
| `onException` | What should happen after routing fails? | Recovery and diagnostics |

I initially handled every unusual route in `redirect`. That made the callback difficult to reason about: a bad ID, a missing permission, and a signed-out user all looked like redirects. Keeping `onException` for failures makes the routing contract much easier to test.

## Add a safe recovery route

The following router sends known routing failures to `/route-error`. The callback receives the `BuildContext`, the `GoRouterState`, and the router instance, so it can inspect the error and start a recovery navigation.

```dart
final router = GoRouter(
  initialLocation: '/home',
  routes: [
    GoRoute(
      path: '/home',
      builder: (context, state) => const HomeScreen(),
    ),
    GoRoute(
      path: '/route-error',
      builder: (context, state) {
        final message = state.extra as String?;
        return RouteErrorScreen(message: message);
      },
    ),
  ],
  onException: (context, state, router) {
    final error = state.error;
    router.go('/route-error', extra: _safeMessage(error));
  },
);

String _safeMessage(Object? error) {
  if (error is GoException) {
    return 'The requested route could not be opened.';
  }
  return 'Navigation failed. Please return to the home screen.';
}
```

The recovery screen should not print a raw exception to a customer. Error text can contain route values, parser details, or implementation names that are useful in logs but confusing in the UI. Keep the detailed value in a logging service and show a stable message to the user.

## Do not create an exception loop

The recovery route must be guaranteed to match. If `/route-error` itself requires authentication, has a malformed builder, or is affected by the same failing redirect, the exception handler can invoke itself again.

```dart
onException: (context, state, router) {
  if (state.matchedLocation == '/route-error') {
    return; // Keep the router's built-in failure behavior.
  }

  _logRoutingFailure(
    error: state.error,
    requestedUri: state.uri,
  );
  router.go('/route-error');
},
```

This guard is not a substitute for a valid error route. It is a last line of defense while the failure is being diagnosed. In production, I also keep the recovery page independent of network calls and authentication-dependent data so it can render when the rest of the app is unhealthy.

## Preserve the original URI for debugging

A recovery redirect is helpful only if the failed request remains observable. Log the original `state.uri` before calling `router.go()`. This matters on Flutter Web, where a user may arrive directly at a copied URL such as `/orders/not-a-number`.

```dart
void _logRoutingFailure({
  required Object? error,
  required Uri requestedUri,
}) {
  debugPrint(
    'go_router failure: uri=$requestedUri error=$error',
  );
}
```

Do not use the recovery route's URI as the diagnostic source; after `router.go('/route-error')`, the router state describes the new location. Capture the failed URI first.

## What `onException` should not do

`onException` is a poor place for ordinary application decisions. A signed-out user should be sent to a login route by `redirect`, and a dirty editor should ask for confirmation through `onExit`. If those cases are routed through exception handling, expected user actions become indistinguishable from actual failures.

There is also a timing trap: the callback is not a general-purpose place to show a dialog and wait for arbitrary UI state. Use it to record the failure and choose a deterministic fallback. If the app needs an asynchronous support report, start that work without delaying recovery.

The practical checklist is short: keep a dependency-light error route, log `state.uri` before recovery navigation, sanitize messages, and protect against a second failure on the recovery path. With those boundaries, `go_router onException` turns broken deep links into recoverable Flutter navigation states instead of blank screens.
