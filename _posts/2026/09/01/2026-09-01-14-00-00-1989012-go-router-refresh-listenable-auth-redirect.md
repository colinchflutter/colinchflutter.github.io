---
layout: post
title: "go_router refreshListenable - Re-run Flutter Auth Redirects Without Manual Navigation"
description: "Learn how go_router refreshListenable re-evaluates Flutter auth redirects, preserves deep links, and avoids redirect loops after login state changes."
date: 2026-09-01
tags: [go_router, navigation, state_management, Flutter, testing]
comments: true
share: true
---

![go_router refreshListenable re-evaluating an authentication redirect in Flutter](/assets/images/go-router-refresh-redirect-auth.png)

`go_router` can re-evaluate a Flutter `redirect` automatically when authentication changes, but only if the router is listening to the same state object that the redirect reads. `refreshListenable` is the small connection that makes this work. It removes the need to call `context.go('/login')` from an auth service and keeps deep-link handling in one place.

## The problem: auth changes after the route is already built

A common first version looks like this:

```dart
final router = GoRouter(
  redirect: (context, state) {
    return auth.isSignedIn ? null : '/login';
  },
  routes: routes,
);
```

The callback is correct for an initial navigation. It is not reactive, though. If a token expires while the user is viewing `/account`, changing `auth.isSignedIn` alone does not tell the router that it should run the callback again. The screen can remain visible even though the auth state says it is protected.

I initially solved this by navigating from the authentication layer. That created two sources of truth: the auth service pushed a route, while the router also had a redirect. Browser back navigation and a cold-start deep link eventually exposed the mismatch.

## Connect a `ChangeNotifier` to the router

`refreshListenable` accepts a `Listenable`. A `ChangeNotifier` is enough for a small auth state object:

```dart
class AuthState extends ChangeNotifier {
  bool _isReady = false;
  bool _isSignedIn = false;

  bool get isReady => _isReady;
  bool get isSignedIn => _isSignedIn;

  void finishLoading({required bool signedIn}) {
    _isReady = true;
    _isSignedIn = signedIn;
    notifyListeners();
  }

  void signIn() {
    _isReady = true;
    _isSignedIn = true;
    notifyListeners();
  }

  void signOut() {
    _isSignedIn = false;
    notifyListeners();
  }
}

final auth = AuthState();
```

The important detail is not the class itself. `notifyListeners()` must run after every state transition that can change the redirect result. Without it, `refreshListenable` has nothing to observe.

## Preserve the requested deep link

The router can now own the complete decision. This example sends an unauthenticated user to `/login`, keeps the original location in a query parameter, and returns the user after a successful sign-in:

```dart
final router = GoRouter(
  refreshListenable: auth,
  redirect: (context, state) {
    final path = state.uri.path;
    final isLogin = path == '/login';
    final isSplash = path == '/splash';
    final isPublic = isLogin || isSplash || path == '/about';

    if (!auth.isReady) {
      return isSplash ? null : '/splash';
    }

    if (!auth.isSignedIn && !isPublic) {
      final from = Uri.encodeComponent(state.uri.toString());
      return '/login?from=$from';
    }

    if (auth.isSignedIn && isLogin) {
      return state.uri.queryParameters['from'] ?? '/';
    }

    return null;
  },
  routes: [
    GoRoute(path: '/splash', builder: (_, __) => const SplashPage()),
    GoRoute(path: '/login', builder: (_, __) => const LoginPage()),
    GoRoute(path: '/about', builder: (_, __) => const AboutPage()),
    GoRoute(path: '/account', builder: (_, __) => const AccountPage()),
  ],
);
```

This produces a predictable flow:

| Event | Router result |
| --- | --- |
| App starts before auth is restored | `/splash` |
| User opens `/account` while signed out | `/login?from=%2Faccount` |
| Login calls `auth.signIn()` | Original `/account` is restored |
| Token expires on `/account` | Redirect returns to `/login` |
| User opens `/about` while signed out | Route remains public |

The `isPublic` check is essential. A redirect that sends `/login` to `/login` can hit go_router's redirect limit and show an error page. The same applies to `/splash` while the initial auth check is still running.

## Avoid the two refresh mistakes I see most often

One mistake is creating the notifier inside `build`. Rebuilding the app can replace the object that the router is listening to, leaving updates attached to an old instance. Create it above `MaterialApp.router`, or provide one stable instance through your state-management layer.

Second, do not put a network request directly in a redirect that runs on every refresh. Redirects may be evaluated more often than expected. Load the auth session in a controller, update the notifier once, and let the synchronous redirect read the resulting state. If the check is genuinely asynchronous, keep the loading state explicit and route to a splash screen while it is pending.

A focused widget test should verify both the redirect and the preserved location:

```dart
testWidgets('auth refresh returns to the requested route', (tester) async {
  final auth = AuthState();
  final router = GoRouter(
    refreshListenable: auth,
    redirect: (context, state) {
      if (!auth.isReady) return '/splash';
      if (!auth.isSignedIn && state.uri.path != '/login') {
        return '/login?from=${Uri.encodeComponent(state.uri.toString())}';
      }
      if (auth.isSignedIn && state.uri.path == '/login') {
        return state.uri.queryParameters['from'] ?? '/';
      }
      return null;
    },
    routes: testRoutes,
  );

  auth.finishLoading(signedIn: false);
  await tester.pumpWidget(MaterialApp.router(routerConfig: router));
  router.go('/account');
  await tester.pumpAndSettle();

  expect(router.state.uri.path, '/login');
  expect(router.state.uri.queryParameters['from'], '/account');
  auth.signIn();
  await tester.pumpAndSettle();
  expect(router.state.uri.path, '/account');
});
```

Adjust the final expectation to the route you actually started with; the useful assertion is that a notifier update causes a new redirect evaluation without a manual `go()` call.

## Short checklist

- Pass the stable auth `Listenable` through `refreshListenable`.
- Call `notifyListeners()` after readiness, sign-in, sign-out, and expiry changes.
- Keep splash and login routes out of the protected-route branch.
- Encode the original `state.uri`, not only `state.uri.path`, when query filters matter.
- Test browser refresh and an expired session, not only a button-driven login.

`refreshListenable` is not a navigation command. It is a signal that tells `go_router` to reconsider the current location. Keeping that distinction clear makes auth redirects work consistently across app startup, deep links, browser history, and session expiry.

Reference: [go_router redirection documentation](https://pub.dev/documentation/go_router/latest/topics/Redirection-topic.html) and [GoRouter API](https://pub.dev/documentation/go_router/latest/go_router/GoRouter-class.html).
