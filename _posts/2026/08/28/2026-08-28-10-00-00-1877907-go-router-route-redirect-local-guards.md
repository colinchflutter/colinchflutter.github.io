---
layout: post
title: "go_router GoRoute.redirect - Keep Flutter Route Guards Local and Predictable"
description: "Learn how to use GoRoute.redirect for route-specific Flutter access rules without turning one global go_router redirect into a maze."
date: 2026-08-28
tags: [go_router, navigation, Flutter, testing]
comments: true
share: true
---

![go_router route guard protecting a Flutter page before navigation](/assets/images/go-router-on-exit-route-guard.png)

`GoRoute.redirect` is a good fit when an access rule belongs to one route branch, not to the whole Flutter application. I use the top-level redirect for session-wide decisions such as “signed-out users must leave the app area,” and a route-level redirect for narrower rules such as “only verified users can open billing.” This split keeps route matching readable and prevents unrelated pages from inheriting conditions they do not need.

## Why a single global redirect becomes difficult to maintain

A global redirect starts innocently:

```dart
GoRouter(
  redirect: (context, state) {
    final signedIn = session.isSignedIn;
    final isLogin = state.matchedLocation == '/login';

    if (!signedIn && !isLogin) return '/login';
    if (signedIn && isLogin) return '/home';
    return null;
  },
  routes: routes,
);
```

The problem appears when the application gains several independent policies. The callback now has to know about onboarding, email verification, subscription status, admin permissions, and maintenance mode. A change for `/billing` can accidentally affect `/settings`, because both decisions are made in the same function.

The practical rule is simple:

| Rule | Best location |
| --- | --- |
| Signed-out versus signed-in | Top-level `GoRouter.redirect` |
| Verification required for one branch | Parent or child `GoRoute.redirect` |
| Confirmation before removing a page | `onExit` |
| Data validation after matching a URL | Route builder or error handling |

## Put a branch-specific policy on `GoRoute`

Suppose `/billing` should be visible only after the account email has been verified. The route can own that decision:

```dart
final router = GoRouter(
  redirect: (context, state) {
    if (!session.isSignedIn && state.matchedLocation != '/login') {
      return '/login';
    }
    return null;
  },
  routes: [
    GoRoute(
      path: '/login',
      builder: (context, state) => const LoginPage(),
    ),
    GoRoute(
      path: '/app',
      builder: (context, state) => const AppPage(),
      routes: [
        GoRoute(
          path: 'billing',
          redirect: (context, state) {
            if (!session.isEmailVerified) return '/app/verify-email';
            return null;
          },
          builder: (context, state) => const BillingPage(),
        ),
        GoRoute(
          path: 'verify-email',
          builder: (context, state) => const VerifyEmailPage(),
        ),
      ],
    ),
  ],
);
```

Returning `null` is not an omission. It explicitly means that this redirect has no objection to the current location. If the user is verified, `/app/billing` continues to its builder. If not, only the billing branch changes the destination.

The returned location is absolute in this example. That makes the result unambiguous when the route is nested. A relative-looking value such as `verify-email` is easy to misread and can produce a location that is not the one intended by the route tree.

## Preserve the requested destination carefully

Redirecting to a verification page is useful only if the user can return to the page they originally wanted. A query parameter is enough for a small, URL-safe destination:

```dart
GoRoute(
  path: 'billing',
  redirect: (context, state) {
    if (!session.isEmailVerified) {
      final target = state.uri.path;
      return Uri(
        path: '/app/verify-email',
        queryParameters: {'returnTo': target},
      ).toString();
    }
    return null;
  },
  builder: (context, state) => const BillingPage(),
),
```

Do not copy an arbitrary browser-provided URL into a later navigation call. Restrict the value to internal paths before using it:

```dart
String? safeReturnPath(String? value) {
  if (value == null || !value.startsWith('/app/')) return null;
  if (value.contains('://') || value.startsWith('//')) return null;
  return value;
}
```

The verification page can then use the checked value:

```dart
final returnTo = safeReturnPath(state.uri.queryParameters['returnTo']);

onVerified: () {
  context.go(returnTo ?? '/app');
},
```

This is intentionally conservative. The return path is navigation input, not just display text, so accepting external URLs or protocol-relative values can create an open-redirect problem.

## Avoid redirect loops at the route boundary

The destination of a route-level redirect must be outside the condition that triggered it. If `/billing` redirects to `/app/verify-email`, the verification route must not itself require `isEmailVerified`:

```dart
GoRoute(
  path: 'verify-email',
  redirect: (context, state) {
    return session.isEmailVerified ? '/app' : null;
  },
  builder: (context, state) => const VerifyEmailPage(),
),
```

This second redirect is safe because its condition is the opposite of the page's purpose. After verification, it removes the now-obsolete page; before verification, it returns `null` and lets the page render.

Another common mistake is checking only a broad prefix:

```dart
// Too broad: this also catches /app/verify-email.
if (!session.isEmailVerified && state.matchedLocation.startsWith('/app')) {
  return '/app/verify-email';
}
```

Match the actual protected branch or explicitly allow the escape route. `state.matchedLocation` is useful when the rule is about the matched route tree; `state.uri` is better when query parameters are part of the decision.

## Refresh the router from the same state source

A route-level redirect is evaluated during routing. It does not continuously watch a session object by itself. If verification changes while the user remains in the app, the router needs a notification source, such as the `refreshListenable` used by the top-level router:

```dart
final router = GoRouter(
  refreshListenable: session,
  redirect: (context, state) {
    if (!session.isSignedIn && state.matchedLocation != '/login') {
      return '/login';
    }
    return null;
  },
  routes: routes,
);
```

The route-level callback reads the same `session` instance. Mixing a provider's cached value, a separate singleton, and a widget-local boolean makes the result timing-dependent. In my tests, the most confusing failures came from a session update that rebuilt the page but did not cause the router to re-evaluate its redirect.

## Test the branch, not just the final widget

A useful test matrix checks the policy boundary directly:

| Session state | Location | Expected result |
| --- | --- | --- |
| Signed out | `/app/billing` | `/login` from the global guard |
| Signed in, unverified | `/app/billing` | `/app/verify-email` from the route guard |
| Signed in, verified | `/app/billing` | Billing page renders |
| Signed in, unverified | `/app/verify-email` | Verification page renders |
| Newly verified | `/app/verify-email` | Redirects to a safe internal destination |

The important assertion is which redirect owns the decision. That detail catches accidental policy movement during refactoring. It also makes failures easier to diagnose than a generic “wrong page was displayed” widget test.

`GoRoute.redirect` works best as a small, local contract: inspect the current route, return a safe absolute location when the branch is not available, and return `null` when the branch is valid. Keep authentication at the router boundary, keep specialized permissions near their routes, and make every redirect destination an intentional escape from its own condition.
