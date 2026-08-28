---
layout: post
title: "go_router GoRouteData.redirect - Add Type-Safe Flutter Route Guards"
description: "Learn how to use GoRouteData.redirect with go_router_builder to keep Flutter authentication and role checks type-safe and loop-free."
date: 2026-08-28
tags: [go_router, navigation, Flutter, testing]
comments: true
share: true
---

![go_router typed route redirect protecting a Flutter premium page](/assets/images/go-router-extra-typed-route-data.png)

`GoRouteData.redirect` is a clean place for a rule that belongs to one typed route. With `go_router_builder`, the route declaration, generated location, and access check can stay together instead of growing another branch in a global redirect callback. I use a global redirect for application-wide session rules, then let a typed route decide whether its own feature is available.

## The problem with putting every permission in one redirect

A top-level redirect often starts with an authentication check and ends up handling onboarding, subscription tiers, workspace membership, and email verification. The callback still returns `String?`, but the meaning of each string becomes difficult to track.

| Rule | Best location | Why |
| --- | --- | --- |
| Every private route requires a session | `GoRouter.redirect` | One policy affects the whole app |
| Only members can open `/premium` | `GoRouteData.redirect` | The rule belongs to one route |
| A product ID must be parsed before display | `build` or typed constructor | It is route input validation |

The typed redirect also gets the route's `GoRouterState`. That matters when the login screen needs to return the user to the exact deep link, including its query string.

## A typed route with a local access check

The following example assumes an `AuthState.of(context)` helper that exposes the current user. The important part is that the route returns `null` when navigation is allowed and a location when it is not.

```dart
import 'package:flutter/material.dart';
import 'package:go_router/go_router.dart';
import 'package:go_router_builder/go_router_builder.dart';

part 'app_routes.g.dart';

@TypedGoRoute<PremiumRoute>(path: '/premium')
class PremiumRoute extends GoRouteData {
  const PremiumRoute();

  @override
  String? redirect(BuildContext context, GoRouterState state) {
    final auth = AuthState.of(context);

    if (!auth.isSignedIn) {
      return Uri(
        path: '/sign-in',
        queryParameters: <String, String>{
          'from': state.uri.toString(),
        },
      ).toString();
    }

    if (!auth.isPremium) {
      return const UpgradeRoute().location;
    }

    return null;
  }

  @override
  Widget build(BuildContext context, GoRouterState state) {
    return const PremiumScreen();
  }
}

@TypedGoRoute<UpgradeRoute>(path: '/upgrade')
class UpgradeRoute extends GoRouteData {
  const UpgradeRoute();

  @override
  Widget build(BuildContext context, GoRouterState state) {
    return const UpgradeScreen();
  }
}
```

There are two useful details here. `state.uri.toString()` preserves the full requested URL, while `Uri` performs the query-string encoding. Concatenating `'?from=$location'` looks shorter, but breaks as soon as the original location already contains `?`, `&`, or a space.

The generated `UpgradeRoute.location` is safer than hard-coding `/upgrade`. If the path changes later, the compiler-generated route location changes with it. For the sign-in destination, a `Uri` is still convenient because `from` is dynamic data rather than a fixed typed route argument.

## Reading and validating the return location

The sign-in route should not blindly call `context.go(from)`. A URL received from a query parameter can be external, so accepting it creates an open redirect. Keep the boundary small and allow only an internal path.

```dart
String safeInternalLocation(String? value) {
  if (value == null || value.isEmpty) return '/';

  final uri = Uri.tryParse(value);
  if (uri == null || uri.hasScheme || uri.host.isNotEmpty) {
    return '/';
  }

  return uri.path.startsWith('/') ? uri.toString() : '/';
}

class SignInRoute extends GoRouteData {
  const SignInRoute({this.from});

  final String? from;

  @override
  Widget build(BuildContext context, GoRouterState state) {
    final returnLocation = safeInternalLocation(from);

    return SignInScreen(
      onSignedIn: () => context.go(returnLocation),
    );
  }
}
```

In a real generated route, declare `from` as the route's query parameter according to the `go_router_builder` version in `pubspec.yaml`. The generated API has changed around annotations in different releases, so I keep that declaration close to the route and verify the generated `.g.dart` after upgrading the package.

## Traps that cause redirect loops

The most common failure is protecting the destination route as well as the original route. If `UpgradeRoute.redirect` sends users back to `PremiumRoute`, the router can bounce between the two. Public routes such as sign-in and upgrade must be outside the same guard condition.

Another trap is reading a mutable auth object once and expecting the redirect to run again automatically. When authentication changes, the router needs a refresh mechanism such as `refreshListenable`, or a `GoRouter` rebuild driven by the auth state. A correct redirect function cannot react to state changes that never trigger route evaluation.

For testing, cover the three outcomes independently:

- signed out → `/sign-in?from=...`
- signed in without a premium plan → `/upgrade`
- signed in with premium access → `null`

`GoRouteData.redirect` works best as a narrow policy boundary: keep global session behavior at the router level, keep feature permissions beside the typed route, and validate every dynamic return location before using it.
