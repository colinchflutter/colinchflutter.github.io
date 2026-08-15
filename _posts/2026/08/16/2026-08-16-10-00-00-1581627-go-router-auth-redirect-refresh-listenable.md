---
layout: post
title: "go_router Authentication Redirect - Protect Flutter Routes with refreshListenable"
description: "Use go_router redirect and refreshListenable to protect Flutter routes, preserve the requested URL, and avoid redirect loops when auth state changes."
date: 2026-08-16
tags: [go_router, navigation, state_management, Flutter]
comments: true
share: true
---

![Flutter app navigation flow from a protected route to login and back](https://images.unsplash.com/photo-1551650975-87deedd944c3?w=1200&q=80)

*The important part is the route decision: an unauthenticated user should reach login, then return to the exact protected URL they originally requested.*

`go_router` authentication is easiest to reason about when the router owns the redirect decision and an auth object owns only the session state. The missing piece in many implementations is `refreshListenable`: without it, changing `isSignedIn` does not necessarily cause the current route to be evaluated again.

## The problem with checking auth inside every screen

I initially put an `if (!auth.isSignedIn)` check in each protected page. That looked harmless, but deep links exposed two problems:

1. The app briefly built a protected screen before replacing it with login.
2. After signing in, the user landed on `/` instead of the deep link they opened.

The router already sees every location, so it is a better boundary for this rule. Keep the redirect small and make the auth object observable.

## A `ChangeNotifier` that the router can observe

This example uses an in-memory session. A real app can replace the setter with a Firebase, OAuth, or secure-token callback while keeping the `notifyListeners()` contract.

```dart
class AuthState extends ChangeNotifier {
  bool _isSignedIn = false;

  bool get isSignedIn => _isSignedIn;

  void signIn() {
    _isSignedIn = true;
    notifyListeners();
  }

  void signOut() {
    _isSignedIn = false;
    notifyListeners();
  }
}
```

The notification is not a navigation command. It only tells `GoRouter` to run its redirect callback again. That separation matters when a token refresh, logout button, or app-start session restore changes the same state.

## Protect routes and preserve the original location

Here is a complete router configuration. The `from` query parameter carries the original path and query string through the login page.

```dart
final auth = AuthState();

final router = GoRouter(
  refreshListenable: auth,
  initialLocation: '/',
  redirect: (context, state) {
    final signedIn = auth.isSignedIn;
    final isLogin = state.matchedLocation == '/login';

    if (!signedIn && !isLogin) {
      final from = Uri.encodeComponent(state.uri.toString());
      return '/login?from=$from';
    }

    if (signedIn && isLogin) {
      return state.uri.queryParameters['from'] ?? '/';
    }

    return null;
  },
  routes: [
    GoRoute(
      path: '/',
      builder: (context, state) => const HomePage(),
    ),
    GoRoute(
      path: '/settings',
      builder: (context, state) => const SettingsPage(),
    ),
    GoRoute(
      path: '/login',
      builder: (context, state) => LoginPage(
        from: state.uri.queryParameters['from'],
      ),
    ),
  ],
);
```

`return null` means that the current location is allowed to continue. Returning `/login` from a protected route changes the URL declaratively; it is not the same as calling `context.push`, so the protected page does not remain as an unwanted page underneath the login flow.

## Complete the login flow without pushing a second page

The login page only changes auth state. `notifyListeners()` triggers the router, which sees that `/login` is now allowed and returns the saved location.

```dart
class LoginPage extends StatelessWidget {
  const LoginPage({super.key, this.from});

  final String? from;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Center(
        child: FilledButton(
          onPressed: () => auth.signIn(),
          child: const Text('Sign in'),
        ),
      ),
    );
  }
}
```

The `from` field is useful for displaying context such as “Continue to settings,” but navigation should still be decided by the router. Keeping two competing calls—`auth.signIn()` and `context.go(from)`—can race with the redirect callback and produce duplicate history entries.

## Redirect rules worth testing

| Situation | Expected result | Common mistake |
|---|---|---|
| Signed out, opening `/settings` | `/login?from=%2Fsettings` | Hard-coding `/` after login |
| Signed out, opening `/login` | Stay on `/login` | Redirecting every non-auth state |
| Signed in, opening `/login` | Saved path or `/` | Returning the encoded value without decoding |
| Signed in, opening `/settings` | Stay on `/settings` | Calling `context.go` from `redirect` |
| User taps sign out | Router moves to login | Updating a boolean without notifying listeners |

There are two traps I would check immediately. First, do not redirect `/login` back to itself when the session is missing; the `isLogin` guard prevents that loop. Second, do not store an arbitrary external URL in `from`. Only restore paths generated by your own app, or validate the value before returning it.

## Practical checklist

- Put session state in a `Listenable`, usually a small `ChangeNotifier`.
- Pass that object through `refreshListenable`.
- Return `null` for allowed locations and a path for redirects.
- Preserve `state.uri`, not only `state.matchedLocation`, when query parameters matter.
- Keep login and logout as state changes; let the router perform the resulting navigation.
- Test cold-start deep links, refresh callbacks, logout, and repeated login taps.

The reliable mental model is simple: `go_router` evaluates a URL, `AuthState` reports whether it is allowed, and `refreshListenable` connects a session change to a fresh evaluation. Once those responsibilities stay separate, protected Flutter routes stop depending on timing and deep links behave like normal URLs.

References: [go_router redirection](https://pub.dev/documentation/go_router/latest/topics/Redirection-topic.html), [go_router navigation](https://pub.dev/documentation/go_router/latest/topics/Navigation-topic.html).
