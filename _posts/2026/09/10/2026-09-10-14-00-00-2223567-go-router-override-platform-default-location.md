---
layout: post
title: "go_router overridePlatformDefaultLocation - Control Flutter App Startup URLs"
description: "Learn how go_router overridePlatformDefaultLocation changes Flutter startup behavior when a platform deep link competes with initialLocation."
date: 2026-09-10
tags: [go_router, navigation, Web, Android]
comments: true
share: true
---

![go_router controlling Flutter startup between an app default and a platform deep link](/assets/images/go-router-stateful-shell-route.png)

`go_router` normally gives a platform-provided location priority over `initialLocation`. That is the behavior a deep-linking app usually needs: opening `/orders/42` should not silently send the user to `/home`. The `overridePlatformDefaultLocation` option changes this rule. When it is `true`, the router uses the configured `initialLocation` even when the platform supplies another startup location.

This is a startup policy, not a redirect replacement. I use it for apps that must always begin in a controlled entry flow, such as a kiosk, a test harness, or a Flutter Web shell embedded at a known route. I do not enable it casually in a public web app, because it can make a copied deep link appear to be ignored.

## `initialLocation` and a platform location are different inputs

The confusing part is that both values look like “the initial route,” but they come from different owners.

| Input | Owner | Example | Normal priority |
| --- | --- | --- | --- |
| `initialLocation` | Application configuration | `/home` | Used when no platform location exists |
| Platform default location | Browser, operating system, or embedder | `/orders/42` | Takes priority on startup |
| `overridePlatformDefaultLocation: true` | Application policy | `/home` | Forces the configured initial location |

The distinction matters on Flutter Web. A browser refresh at `/orders/42` supplies that path to the router before the app has rendered its first page. On Android or iOS, a launch intent or universal link can play the same role. `initialLocation` is the fallback; it is not automatically a command to discard an incoming deep link.

## Keep the default behavior for deep-linkable applications

This is the configuration I would start with for a normal product app. The user can open the home page normally, while a browser URL or external link can select a more specific route.

```dart
final router = GoRouter(
  initialLocation: '/home',
  routes: [
    GoRoute(
      path: '/home',
      builder: (context, state) => const HomePage(),
    ),
    GoRoute(
      path: '/orders/:orderId',
      builder: (context, state) => OrderPage(
        orderId: state.pathParameters['orderId']!,
      ),
    ),
  ],
);
```

If the browser is already at `/orders/42`, the platform location is the useful input and the order page can load `42`. If there is no incoming location, `/home` becomes the starting route. A redirect can still protect `/orders/:orderId` after this initial match; overriding the platform default is not an authentication mechanism.

## Force a known entry route when that is the actual requirement

An onboarding demo, a kiosk display, or a deterministic widget-test host may need every launch to begin at `/welcome`. In that case the option makes the decision explicit:

```dart
final router = GoRouter(
  initialLocation: '/welcome',
  overridePlatformDefaultLocation: true,
  routes: [
    GoRoute(
      path: '/welcome',
      builder: (context, state) => const WelcomePage(),
    ),
    GoRoute(
      path: '/dashboard',
      builder: (context, state) => const DashboardPage(),
    ),
  ],
);
```

With this policy, a platform-supplied `/dashboard` does not win during router initialization. The app starts at `/welcome`, and the user can later navigate to `/dashboard` through an ordinary `go`, `push`, or named navigation call.

The option does not make `/dashboard` unreachable. It only changes the location used for the initial router configuration. That is why it belongs beside `initialLocation`, not inside every route builder.

## Do not use it to solve authentication redirects

I initially considered this option for a signed-out user who opened a private deep link. That mixes two separate decisions:

1. Which location did the platform request?
2. Is the current session allowed to display it?

The first is startup precedence. The second is route policy. Keep the platform location, then use a redirect to protect it and preserve a validated return path.

```dart
String? redirect(BuildContext context, GoRouterState state) {
  final isPublic = state.matchedLocation == '/sign-in';

  if (!auth.isReady) {
    return '/loading';
  }
  if (!auth.isSignedIn && !isPublic) {
    final returnTo = state.uri.toString();
    return Uri(
      path: '/sign-in',
      queryParameters: {'returnTo': returnTo},
    ).toString();
  }
  return null;
}
```

The exact auth implementation will vary, but the boundary is stable: `overridePlatformDefaultLocation` chooses the startup source, while `redirect` evaluates access. If the app always overrides the deep link and then redirects from `/welcome`, the original destination may be lost before the auth policy can inspect it.

## Shell routes make the choice more visible

A `StatefulShellRoute` can preserve several branch navigators, but it does not change startup precedence. The platform may request `/settings/profile`, and the router can still select the settings branch and its nested page. If `overridePlatformDefaultLocation` sends the app to `/home`, the settings branch will not be the initial visible branch unless application code navigates there later.

That difference is easy to miss when testing only by tapping tab buttons. I test both forms of launch:

| Test | Expected result with `false` | Expected result with `true` |
| --- | --- | --- |
| No platform deep link | `initialLocation` | `initialLocation` |
| Launch at `/settings/profile` | Settings profile | Configured initial route |
| Browser refresh on a detail URL | Detail URL | Configured initial route |
| Later `context.go('/settings')` | Settings route | Settings route |

The last row is important. A forced initial location is not a global navigation lock. If later navigation behaves differently, look for a redirect, a shell branch index change, or a widget that rebuilds the router.

## Common traps

- **Expecting `initialLocation` to override a browser URL by default.** It is normally a fallback when the platform has no location.
- **Using the option as a security boundary.** A user can still navigate to a route after startup. Keep authorization in `redirect`, `onEnter`, or the page’s data policy.
- **Rebuilding `GoRouter` on every auth or widget update.** That can reset Navigator state and make startup behavior look like a routing bug. Keep the router stable and use `refreshListenable` for redirect re-evaluation.
- **Enabling it on a shareable Flutter Web site without a product decision.** A user who copies `/orders/42` expects that URL to open the order, not the marketing home page.
- **Testing only a hot restart.** A hot restart does not represent every browser refresh, external link, or platform launch intent. Test a real startup with and without an incoming location.

The useful rule is simple: leave `overridePlatformDefaultLocation` false when URLs are part of the product contract. Set it to true only when the app intentionally owns the first screen regardless of the platform request, and document the decision beside the router configuration.
