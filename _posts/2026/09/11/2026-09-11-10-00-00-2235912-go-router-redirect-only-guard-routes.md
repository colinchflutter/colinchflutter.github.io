---
layout: post
title: "go_router redirectOnly - Build Flutter Guard Routes Without Pages"
description: "Learn how go_router redirectOnly creates guard-only Flutter routes for clean authentication and permission checks without rendering placeholder pages."
date: 2026-09-11
tags: [go_router, navigation, Android, Web]
comments: true
share: true
---

![go_router redirectOnly route acting as a Flutter navigation guard](/assets/images/go-router-on-exit-route-guard.png)

`go_router`'s `redirectOnly` option is useful when a path exists to make a decision, not to display a screen. It turns a `GoRoute` into a guard-only route: the route can match an incoming location and run its redirect callback, but it does not need a `builder` or `pageBuilder`.

I reach for this pattern when a product has a meaningful checkpoint such as `/account`, `/admin`, or `/checkout`. The checkpoint can inspect authentication or permissions and then send the user to the correct real page. The route configuration stays readable because the decision is represented in the route tree instead of hidden inside a page that never appears.

## A normal redirect route still looks like a page route

Without `redirectOnly`, a redirect route is easy to misread. It has a path and a redirect, but the router still expects the route to be renderable in configurations where the redirect does not return another location.

| Route type | Has a screen | Main responsibility | Good use |
| --- | --- | --- | --- |
| Regular `GoRoute` | Yes | Match and render | `/settings` |
| Regular route with `redirect` | Usually | Match, decide, then render if allowed | A page-level guard |
| `redirectOnly: true` | No | Match and redirect | Auth or permission checkpoint |

The important distinction is that a redirect-only route is not a hidden page. It is an explicit routing node whose job ends after it produces another location.

## Basic authentication checkpoint

The following example keeps `/account` as a stable entry URL. Signed-in users continue to the account page; signed-out users go to `/login` with the attempted location preserved.

Put the guard route in the same parent route list as the destinations it can redirect to.

```dart
final router = GoRouter(
  initialLocation: '/home',
  routes: [
    GoRoute(
      path: '/home',
      builder: (context, state) => const HomePage(),
    ),
    GoRoute(
      path: '/account',
      redirectOnly: true,
      redirect: (context, state) {
        final signedIn = AuthState.of(context).isSignedIn;
        if (signedIn) {
          return '/account/details';
        }

        final from = Uri.encodeComponent(state.uri.toString());
        return '/login?from=$from';
      },
    ),
    GoRoute(
      path: '/account/details',
      builder: (context, state) => const AccountPage(),
    ),
    GoRoute(
      path: '/login',
      builder: (context, state) => LoginPage(
        returnLocation: state.uri.queryParameters['from'],
      ),
    ),
  ],
);
```

The callback returns a `String` location when navigation must change and `null` when no redirect is needed. In this example, the signed-in branch still redirects because `/account` is a checkpoint rather than a destination.

There is a practical trap here: `state.uri.toString()` contains the full location, including query parameters. Encoding it before placing it inside `from` prevents the original `?` and `&` characters from becoming part of the login route's own query parameters. After a successful login, the app can decode the value and call `context.go(returnLocation)`.

## Permission checks belong in the same place

Authentication and authorization are separate decisions. A user may be signed in but still lack access to an admin area. A redirect-only route makes that distinction visible.

```dart
GoRoute(
  path: '/admin',
  redirectOnly: true,
  redirect: (context, state) {
    final auth = AuthState.of(context);

    if (!auth.isSignedIn) {
      return '/login?from=%2Fadmin';
    }

    if (!auth.canManageUsers) {
      return '/forbidden';
    }

    return '/admin/users';
  },
),
GoRoute(
  path: '/admin/users',
  builder: (context, state) => const UserAdminPage(),
),
GoRoute(
  path: '/forbidden',
  builder: (context, state) => const ForbiddenPage(),
),
```

This is cleaner than creating an `AdminEntryPage` that runs a redirect during `build`. A widget should not be mounted briefly just to decide that it should never be shown. The redirect callback also runs during route matching, so deep links and browser refreshes use the same policy as button taps.

## Avoid loops and unstable state

`redirectOnly` does not protect against a redirect loop. The target location must eventually match a route that either renders or returns `null`.

```dart
GoRoute(
  path: '/login',
  redirectOnly: true,
  redirect: (context, state) {
    return AuthState.of(context).isSignedIn ? '/home' : null;
  },
),
```

This looks reasonable, but it creates an awkward URL: a signed-out user can stay on `/login`, while a signed-in user is redirected away. That can be intentional for a login entry point, but it should not redirect back to `/login` from another guard. For example, `/account` → `/login` → `/account` will loop if the login route redirects without a real authentication state change.

Keep these checks in mind:

| Check | Why it matters |
| --- | --- |
| Target route exists | A typo becomes a route error instead of a useful checkpoint |
| Auth state is stable | Rebuilding an `InheritedWidget` can re-evaluate redirects |
| Login and forbidden pages are renderable | They need to terminate the redirect chain |
| Return locations are encoded | Nested query strings otherwise get corrupted |
| Redirect targets do not point back to the guard | Prevents redirect-limit failures |

For authentication that changes asynchronously, connect the router to a `refreshListenable` or another notifier already used by the app. The guard should read the current state and return a location; it should not perform login, token refresh, or database writes inside the redirect callback.

## When `redirectOnly` is the wrong fit

Do not use it when the route itself has meaningful content. A loading page, an access explanation, or a form needs a normal `GoRoute` with a `builder` or `pageBuilder`. Also avoid duplicating a global policy in many guard routes when every route follows the same rule; a top-level `GoRouter.redirect` may be easier to audit.

`redirectOnly` fits best when the URL is a named checkpoint in the product flow. It gives that checkpoint a place in the route tree without pretending that it is a visual page.

The short version is:

1. Add `redirectOnly: true` when a route should never render.
2. Return a concrete location for auth or permission decisions, and `null` only when the route is allowed to finish matching.
3. Keep the real destination, login page, and forbidden page as separate renderable routes.
4. Encode preserved locations and test deep links, refreshes, and state changes to catch loops.
