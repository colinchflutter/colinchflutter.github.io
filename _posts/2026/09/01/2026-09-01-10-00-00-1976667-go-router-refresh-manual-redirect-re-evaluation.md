---
layout: post
title: "go_router refresh - Re-evaluate Flutter Redirects After Auth Changes"
description: "Learn how GoRouter.refresh re-evaluates Flutter redirects after authentication changes without rebuilding the router or losing navigation state."
date: 2026-09-01
tags: [go_router, navigation, state_management, testing, Flutter]
comments: true
share: true
---

![go_router manually refreshing Flutter redirect state](/assets/images/go-router-stateful-shell-route.png)

`GoRouter.refresh()` is useful when an authentication or permission change happens outside the router. It re-runs the router's redirect pipeline at the current location, so a signed-out user can leave a protected page immediately without reconstructing the `GoRouter` instance or throwing away the navigation stack.

## The problem with changing auth state alone

An auth controller can notify its widgets perfectly while the router remains unaware of the change. A user signs out, the profile screen rebuilds, and the protected URL is still active because no navigation decision was requested.

| Situation | What should happen | Router action |
|---|---|---|
| Session changes outside a page | Re-check the current URL | `router.refresh()` |
| User taps a destination | Navigate to a new URL | `context.go()` |
| Route must be replaced by policy | Map one location to another | `redirect` |

The important distinction is that `refresh()` is not a page reload. It asks `go_router` to evaluate its current location again. It does not mean “create a new router.”

## Keep the router stable

The redirect callback should read the current auth state, while the session service owns the state change. The router is created once and receives an explicit refresh signal:

```dart
final auth = AuthState();
late final GoRouter router;

router = GoRouter(
  initialLocation: '/dashboard',
  refreshListenable: auth,
  redirect: (context, state) {
    final onSignIn = state.matchedLocation == '/sign-in';

    if (!auth.isSignedIn && !onSignIn) {
      return '/sign-in?from=${Uri.encodeComponent(state.uri.toString())}';
    }
    if (auth.isSignedIn && onSignIn) {
      return '/dashboard';
    }
    return null;
  },
  routes: [
    GoRoute(path: '/sign-in', builder: (_, __) => const SignInPage()),
    GoRoute(path: '/dashboard', builder: (_, __) => const DashboardPage()),
  ],
);
```

With `refreshListenable`, the router calls the redirect again whenever `AuthState` notifies listeners. That is usually cleaner than calling `router.refresh()` from every login and logout button. The manual method still matters when the state source cannot implement `Listenable`, or when a one-off external event must trigger a re-check.

```dart
Future<void> signOut() async {
  await authApi.revokeToken();
  auth.clear();
  router.refresh();
}
```

Do not create a new `GoRouter` inside `build()` to achieve the same effect. That approach can reset page identity, duplicate observers, and make a `StatefulShellRoute` appear to lose its branch state. A stable router plus a refresh signal keeps the navigation owner in one place.

## Preserve the original destination safely

Passing the current URI through a query parameter is convenient, but it must be treated as untrusted input. Keep the value small, encode it, and validate it after sign-in. In particular, do not redirect to arbitrary external URLs or blindly accept a path that has become invalid after a route change.

```dart
String safeReturnPath(String? raw) {
  if (raw == null || raw.isEmpty) return '/dashboard';
  final uri = Uri.tryParse(raw);
  if (uri == null || !uri.path.startsWith('/')) return '/dashboard';
  return uri.path + (uri.hasQuery ? '?${uri.query}' : '');
}
```

A sign-in page can call `context.go(safeReturnPath(state.uri.queryParameters['from']))` after the session is ready. The redirect runs again because the auth object notified listeners; the sign-in route then becomes invalid and the protected destination is allowed.

## Refresh is not a redirect-loop fix

The callback must return `null` for stable states. If `/sign-in` always redirects to `/dashboard` while the session is still signed out, every refresh repeats the same decision until `redirectLimit` stops it. A refresh exposes that bug quickly because it runs the same pipeline without a user tap to hide the cause.

I test both transitions explicitly:

```dart
testWidgets('logout refreshes the current route into sign-in', (tester) async {
  final auth = AuthState()..isSignedIn = true;
  final router = buildRouter(auth);

  await tester.pumpWidget(App(router: router));
  router.go('/dashboard');
  await tester.pumpAndSettle();

  auth.isSignedIn = false;
  router.refresh();
  await tester.pumpAndSettle();

  expect(find.byType(SignInPage), findsOneWidget);
});
```

The test should also cover a deep link, an already active `/sign-in`, and a refresh while a shell branch is selected. Those cases catch accidental stack resets and redirects that only work from the dashboard.

## Practical rules

- Prefer `refreshListenable` for a long-lived auth or permission source.
- Use `router.refresh()` for an external event that has no listenable adapter.
- Keep `GoRouter` outside widget `build()` methods.
- Return `null` when the current location already satisfies the policy.
- Preserve only validated internal paths after sign-in.

The useful mental model is simple: `GoRouter.refresh()` reopens the routing decision, not the application. It is a small tool, but it closes the gap between an external session change and the URL that the user is currently viewing.
