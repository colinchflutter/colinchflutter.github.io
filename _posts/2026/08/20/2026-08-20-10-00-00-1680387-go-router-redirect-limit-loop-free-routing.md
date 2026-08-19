---
layout: post
title: "go_router redirectLimit - Prevent Flutter Redirect Loops Before They Crash"
description: "Learn why go_router redirect loops happen in Flutter and how to use matchedLocation, refreshListenable, and redirectLimit safely."
date: 2026-08-20
tags: [go_router, navigation, state_management, testing]
comments: true
share: true
---

![go_router redirect flow preventing a Flutter navigation loop](/assets/images/go-router-stateful-shell-route.png)

`go_router` redirect loops usually come from a redirect that sends the router to a location it still considers protected. Checking the current matched location before returning a new path fixes the loop; increasing `redirectLimit` only hides the design problem for a little longer.

## The failure pattern

An authentication guard often starts with a simple rule: unauthenticated users go to `/sign-in`, and authenticated users go to `/home`. The trap appears when the guard redirects every location without excluding the destination itself.

| Situation | Unsafe result | Safe result |
|---|---|---|
| Guest opens `/settings` | Redirect to `/sign-in` repeatedly | Redirect once, then return `null` |
| Guest opens `/sign-in` | Redirect to `/sign-in` again | Keep the sign-in route |
| User opens `/sign-in` | Redirect without checking state | Move to `/home` only when appropriate |

The router follows redirects until the location is stable. The default `redirectLimit` is 5, so a loop normally ends as a routing error instead of rendering the page the user expected.

## A loop-free authentication redirect

The auth object must notify the router when its state changes. `refreshListenable` causes the redirect callback to run again after login or logout.

```dart
class AuthController extends ChangeNotifier {
  bool _signedIn = false;

  bool get signedIn => _signedIn;

  void signIn() {
    _signedIn = true;
    notifyListeners();
  }

  void signOut() {
    _signedIn = false;
    notifyListeners();
  }
}

final auth = AuthController();

final router = GoRouter(
  refreshListenable: auth,
  redirectLimit: 5,
  redirect: (context, state) {
    final location = state.matchedLocation;
    final isSignIn = location == '/sign-in';

    if (!auth.signedIn && !isSignIn) {
      return '/sign-in';
    }

    if (auth.signedIn && isSignIn) {
      return '/home';
    }

    return null;
  },
  routes: [
    GoRoute(
      path: '/sign-in',
      builder: (context, state) => const SignInPage(),
    ),
    GoRoute(
      path: '/home',
      builder: (context, state) => const HomePage(),
    ),
    GoRoute(
      path: '/settings',
      builder: (context, state) => const SettingsPage(),
    ),
  ],
);
```

There are two deliberate details here. `isSignIn` is checked before returning `/sign-in`, and every branch that does not need navigation returns `null`. That makes the redirect idempotent: running it again with the same auth state produces the same location.

## Keep the intended URL after login

For a deep link such as `/settings`, redirecting to `/sign-in` should preserve the original destination. A query parameter is enough for a small app.

```dart
if (!auth.signedIn && !isSignIn) {
  final from = Uri.encodeComponent(state.uri.toString());
  return '/sign-in?from=$from';
}
```

The sign-in page can read `state.uri.queryParameters['from']`, validate that it is an internal path, and call `context.go(from)` after authentication. Do not accept an arbitrary external URL as a post-login destination; that turns a navigation convenience into an open redirect.

## What `redirectLimit` is actually for

`redirectLimit` is a safety fuse, not a loop fix. Keep it near the number of redirects your route design genuinely needs. A value such as 5 catches accidental cycles like `/a → /b → /a`, while a very high value makes the failure slower and harder to diagnose.

A practical test should cover the guard itself rather than tapping through the whole UI:

```dart
test('guest stays on sign-in without redirecting again', () {
  auth.signOut();
  expect(router.configuration, isNotNull);
  // Navigate to /sign-in and assert the matched location remains /sign-in.
});
```

The short checklist is: compare against the matched location, return `null` for stable states, refresh the router from one auth source, preserve deep links only after validating them, and treat `redirectLimit` as a diagnostic boundary.
