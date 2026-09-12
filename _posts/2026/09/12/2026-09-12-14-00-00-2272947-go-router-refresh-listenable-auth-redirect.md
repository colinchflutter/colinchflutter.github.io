---
layout: post
title: "go_router refreshListenable - Re-run Flutter Auth Redirects When Session State Changes"
description: "Learn how go_router refreshListenable re-evaluates Flutter auth redirects, including ChangeNotifier, stream adapters, and refresh-loop traps."
date: 2026-09-12
tags: [go_router, navigation, state_management, testing, Web]
comments: true
share: true
---

![go_router refreshListenable re-evaluating Flutter authentication redirects](/assets/images/go-router-stateful-shell-route.png)

*The important detail is that `refreshListenable` re-runs the router's redirect logic; it does not navigate by itself.*

When a Flutter user's session expires, the router must notice the change even if the user has not tapped anything. `go_router`'s `refreshListenable` is the small connection that makes this work. Attach a `Listenable` that represents authentication state, and the current location is evaluated again. A protected page can then redirect to `/login`, while a signed-in user who is looking at `/login` can be sent to `/home`.

## The problem with checking auth only during navigation

This redirect looks reasonable at first:

```dart
redirect: (context, state) {
  final loggedIn = auth.isLoggedIn;
  if (!loggedIn && state.matchedLocation != '/login') {
    return '/login';
  }
  return null;
},
```

It runs when the router parses a new location. It does not automatically run when `auth.isLoggedIn` changes. That creates a familiar failure: the login screen successfully updates the session, but the app remains on `/login` until another navigation event occurs.

`refreshListenable` fixes the missing trigger. The `redirect` callback remains the policy; the `Listenable` only tells `go_router` when that policy should be checked again.

## A minimal ChangeNotifier setup

Keep the auth object alive for at least as long as the router. The notifier must call `notifyListeners()` after the value used by `redirect` changes.

```dart
class AuthController extends ChangeNotifier {
  bool _isLoggedIn = false;

  bool get isLoggedIn => _isLoggedIn;

  Future<void> signIn() async {
    // Replace this with the real token request.
    _isLoggedIn = true;
    notifyListeners();
  }

  Future<void> signOut() async {
    _isLoggedIn = false;
    notifyListeners();
  }
}

final auth = AuthController();

final router = GoRouter(
  refreshListenable: auth,
  redirect: (context, state) {
    final onLogin = state.matchedLocation == '/login';

    if (!auth.isLoggedIn && !onLogin) return '/login';
    if (auth.isLoggedIn && onLogin) return '/home';
    return null;
  },
  routes: [
    GoRoute(path: '/login', builder: (_, __) => const LoginPage()),
    GoRoute(path: '/home', builder: (_, __) => const HomePage()),
  ],
);
```

The login button only changes the state:

```dart
onPressed: () async {
  await auth.signIn();
},
```

There is no `context.go('/home')` here. Calling both can work, but it duplicates routing decisions and can briefly navigate to a location that the redirect immediately replaces. Let the session state be the single source of truth.

## What gets refreshed?

The refresh is a re-evaluation of the router configuration at the current location. It is not a rebuild of every widget and it is not a request to push a new page onto the Navigator.

| Event | Router behavior | Recommended responsibility |
| --- | --- | --- |
| `notifyListeners()` | Runs `redirect` again | Auth controller publishes state |
| Redirect returns a path | Replaces the current route location | Router owns access policy |
| Redirect returns `null` | Keeps the current location | Page renders normally |
| No state notification | Nothing is re-evaluated | Avoid mutating auth silently |

This distinction matters with token refresh. A background API client may receive a 401 and clear its token, but the UI will stay on the protected page unless that operation also updates the notifier.

## Connecting a stream with GoRouterRefreshStream

Some state-management libraries expose a `Stream` rather than a `Listenable`. `go_router` provides `GoRouterRefreshStream` as an adapter:

```dart
final sessionStream = sessionEvents.map((event) => event.isAuthenticated);

final router = GoRouter(
  refreshListenable: GoRouterRefreshStream(sessionStream),
  redirect: (context, state) {
    final session = sessionStore.current;
    return session.isAuthenticated ? null : '/login';
  },
  routes: routes,
);
```

The callback still needs a synchronous source such as `sessionStore.current`. The stream is the refresh signal; it is not the value that `redirect` receives as an argument. Keep those two roles separate, or the router may re-run before the store has been updated.

## Traps that cause loops and stale auth

A redirect loop usually means the redirect target also fails its own condition. Always exempt public routes such as `/login`, `/forgot-password`, and `/maintenance` from the protected-route rule. Also avoid emitting a notification for an unchanged state. Repeated notifications are safe in small amounts, but they make async redirects harder to reason about and can hit the router's redirect limit when the policy is not stable.

For startup, model the unknown state explicitly. Treating “auth request still loading” as logged out sends users to `/login` and then immediately away when the token is restored. A splash route or an `isLoading` guard gives the router a stable decision.

```dart
redirect: (context, state) {
  if (auth.isLoading) return '/splash';
  if (!auth.isLoggedIn && state.matchedLocation != '/login') {
    return '/login';
  }
  return null;
},
```

## Testing the behavior that matters

Test the state transition, not just whether a button calls `go()`. Build a router with a `MaterialApp.router`, call `signIn()` or `signOut()`, and pump the tester. The visible route should change without an explicit navigation call. This catches the two regressions I see most often: forgetting `notifyListeners()` and accidentally constructing a second auth object for the router.

The practical rule is simple: put access policy in `redirect`, publish auth changes through one long-lived `Listenable`, and let `refreshListenable` connect the two. That keeps login, logout, token expiry, and deep-link protection on the same declarative path.

Reference: [GoRouter API](https://pub.dev/documentation/go_router/latest/go_router/GoRouter-class.html), [GoRouterRefreshStream API](https://pub.dev/documentation/go_router/latest/go_router/GoRouterRefreshStream-class.html), and [Redirection topic](https://pub.dev/documentation/go_router/latest/topics/Redirection-topic.html).
