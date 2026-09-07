---
layout: post
title: "go_router redirectLimit - Debug Flutter Redirect Loops Safely"
description: "Learn how go_router redirectLimit detects Flutter redirect loops, how to fix unstable guards, and how to test navigation without hiding real failures."
date: 2026-09-07
tags: [go_router, navigation, testing, debugging]
comments: true
share: true
---
![go_router redirect loop recovery in Flutter](/assets/images/go-router-on-exit-route-guard.png)

`go_router`'s `redirectLimit` is a safety boundary for Flutter navigation, not a switch that makes a bad redirect correct. When a redirect keeps returning another location instead of reaching a stable route, the router stops after the configured number of redirects and exposes a navigation error. Understanding that boundary makes authentication guards, onboarding flows, and deep-link recovery much easier to debug.

## Why a redirect loop happens

A redirect is stable when it returns `null`, or when it eventually returns a location that no longer matches the redirect condition. A loop appears when the condition never changes. These are common examples:

| Situation | Typical mistake | Result |
| --- | --- | --- |
| Signed-out user visits `/account` | `/login` is not excluded from the guard | `/login` redirects to `/login` |
| User is still loading | Both `/splash` and the protected route redirect | The loading state never settles |
| Missing product ID | Fallback route requires the same ID | Recovery redirects back to failure |
| Two guards disagree | One sends `/home` to `/setup`, the other sends `/setup` to `/home` | Alternating locations |

I initially increased the limit when an authentication flow hit the error page. That only delayed the failure. The useful question is which redirect decision is still true after the location changes.

## Configure `redirectLimit` as a diagnostic boundary

The constructor accepts an integer that controls how many redirects can be followed during one navigation attempt. Keep it finite and reasonably small. A higher value can be useful for a deliberately nested route policy, but a value such as `100` usually hides a cycle rather than supporting a real flow.

Here is a small router with an explicit loading state and a loop-safe authentication guard:

```dart
final auth = AuthState();

final router = GoRouter(
  redirectLimit: 8,
  refreshListenable: auth,
  initialLocation: '/splash',
  redirect: (context, state) {
    final location = state.uri.path;

    if (auth.isLoading) {
      return location == '/splash' ? null : '/splash';
    }

    final onPublicRoute = location == '/login' || location == '/splash';
    if (!auth.isSignedIn && !onPublicRoute) {
      return '/login?from=${Uri.encodeComponent(state.uri.toString())}';
    }

    if (auth.isSignedIn && location == '/login') {
      return '/home';
    }

    return null;
  },
  routes: [
    GoRoute(path: '/splash', builder: (_, __) => const SplashPage()),
    GoRoute(path: '/login', builder: (_, __) => const LoginPage()),
    GoRoute(path: '/home', builder: (_, __) => const HomePage()),
    GoRoute(path: '/account', builder: (_, __) => const AccountPage()),
  ],
);
```

The important detail is that every branch can settle. While the session is loading, `/splash` returns `null`. Once loading finishes, the same current location is evaluated again because `refreshListenable` notified the router. A signed-out `/account` becomes `/login`, and `/login` is explicitly public, so the next evaluation returns `null`.

`redirectLimit` does not count arbitrary calls to `context.go()`. It protects the redirect pipeline from repeatedly transforming one navigation request. That distinction matters when investigating logs: a user tapping two buttons can create separate navigation events, while a redirect cycle happens inside one event.

## Keep redirect targets outside the guard condition

The fastest way to find a loop is to write the route transitions down:

```text
/account  --signed out-->  /login
/login   --signed out-->  null
```

If the second line instead points to `/login`, the guard is not idempotent. The same problem appears in role-based routing:

```dart
String? redirectByRole(GoRouterState state, User user) {
  final path = state.uri.path;

  if (!user.hasProfile && path != '/complete-profile') {
    return '/complete-profile';
  }

  if (user.hasProfile && path == '/complete-profile') {
    return '/home';
  }

  return null;
}
```

The target `/complete-profile` is excluded from the first condition. Without that exception, the router would keep returning the same location until `redirectLimit` terminated the attempt.

Avoid using a redirect to perform the work that decides whether the redirect is needed. Network calls, database reads, and token refreshes make the callback unpredictable and can update the very state that the callback is reading. Load that state in a controller, expose a stable snapshot, then notify the router once the snapshot changes.

## Diagnose the exact cycle

Enable diagnostic logging temporarily while developing the guard:

```dart
final router = GoRouter(
  debugLogDiagnostics: true,
  redirectLimit: 8,
  redirect: appRedirect,
  routes: appRoutes,
);
```

Look for a repeated pair of locations, not only the final exception. A sequence such as `/home → /setup → /home` identifies two conflicting policies. A sequence repeating `/login` points to a missing public-route exception. If only the production build fails, compare the authentication snapshot at the first redirect with the snapshot after `refreshListenable` fires; an eagerly updated notifier can expose a short-lived contradictory state.

Do not catch the redirect-limit error and silently send the user to `/home`. That can turn a broken authorization rule into an accidental access path. A safer recovery page can log the original error and offer a public destination, while the policy itself is fixed in code.

## Test the boundary, not just the happy path

A focused test should prove that a protected deep link settles on the login page and that the public target does not redirect again:

```dart
testWidgets('signed-out deep link settles at login', (tester) async {
  final auth = AuthState()..finishSignedOut();
  final router = createRouter(auth);

  await tester.pumpWidget(
    MaterialApp.router(routerConfig: router),
  );
  router.go('/account?tab=billing');
  await tester.pumpAndSettle();

  expect(router.state.uri.path, '/login');
  expect(router.state.uri.queryParameters['from'],
      '/account?tab=billing');
});
```

Also test the loading route, an already active `/login`, a signed-in user opening `/login`, and a malformed deep link whose fallback is public. These cases catch loops that a simple home-page test misses.

The practical rule is simple: use `redirectLimit` to fail fast and reveal an unstable policy, then make each redirect target explicitly stable. A finite limit, synchronous state reads, public fallback routes, and transition-focused tests keep `go_router` navigation predictable without masking authorization bugs.

Reference: [go_router redirection documentation](https://pub.dev/documentation/go_router/latest/topics/Redirection-topic.html) and [GoRouter API](https://pub.dev/documentation/go_router/latest/go_router/GoRouter-class.html).
