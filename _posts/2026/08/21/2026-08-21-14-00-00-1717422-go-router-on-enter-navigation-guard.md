---
layout: post
title: "go_router onEnter - Block Flutter Navigation Before Redirects Run"
description: "Learn how go_router onEnter blocks unsafe Flutter navigation asynchronously and how it differs from redirect and onExit."
date: 2026-08-21
tags: [go_router, navigation, state_management, testing]
comments: true
share: true
---

![go_router navigation guard flow in Flutter](/assets/images/go-router-stateful-shell-route.png)

`go_router`'s `onEnter` is the right tool when navigation needs a yes-or-no decision before the router processes the destination. Unlike `redirect`, it does not replace the URL with another route. It either returns `Allow()` and lets navigation continue, or returns `Block` and restores the previous route.

## Why `redirect` is not always enough

I initially put every navigation rule in `redirect`. That worked for authentication, but it became awkward for a checkout page that needed an asynchronous confirmation. A redirect must return a location, so a temporary decision such as “the cart is still syncing” gets mixed with URL policy.

These callbacks have different jobs:

| Callback | Decision | Typical use |
| --- | --- | --- |
| `onEnter` | Allow or block | Permission, async preflight, unsaved-work check |
| `redirect` | Return another location | Auth, onboarding, canonical URLs |
| `onExit` | Allow or reject leaving | Route-specific exit confirmation |

`onEnter` receives both the current and incoming `GoRouterState`, which is useful when the rule depends on the destination rather than the page being left.

## Add an asynchronous navigation guard

The current API exposes `onEnter` through `RoutingConfig`, which is passed to `GoRouter.routingConfig`. This example blocks checkout while the cart is empty and allows every other destination.

```dart
import 'package:flutter/material.dart';
import 'package:go_router/go_router.dart';

final routingConfig = ValueNotifier<RoutingConfig>(
  RoutingConfig(
    routes: [
      GoRoute(
        path: '/',
        builder: (context, state) => const HomePage(),
      ),
      GoRoute(
        path: '/checkout',
        builder: (context, state) => const CheckoutPage(),
      ),
    ],
    onEnter: (context, current, next, router) async {
      if (next.uri.path != '/checkout') {
        return const Allow();
      }

      final hasItems = await CartService.instance.hasItems();
      if (hasItems) return const Allow();

      return const Block.stop();
    },
  ),
);

final router = GoRouter.routingConfig(
  routingConfig: routingConfig,
);
```

`Block.stop()` is deliberately a hard stop. It prevents the route change without starting another navigation. If the desired behavior is to send the user somewhere else, use `Block.then` and navigate after the block has been committed:

```dart
return Block.then(() => router.go('/cart'));
```

Do not call `router.go` before returning `Block`. That creates competing navigation requests and can make the browser URL briefly show the blocked destination.

## Keep `onEnter` and `redirect` separate

An authentication rule still belongs in `redirect` because it maps one location to another. A payment permission check belongs in `onEnter` when the correct outcome is simply to stay where the user is. A useful split looks like this:

```dart
redirect: (context, state) {
  final signedIn = AuthService.instance.isSignedIn;
  if (!signedIn && state.uri.path != '/login') return '/login';
  return null;
},
```

The router evaluates `onEnter` before the legacy top-level `redirect`. That means an `onEnter` guard can stop navigation before redirect logic runs, but it should not be used as a hidden replacement for an authentication redirect.

## Initial deep links need a fallback

There may be no previous route to restore when an app launches directly at a blocked URL. In that case `go_router` can report a `BlockedInitialNavigationException`. Handle that specific error in `onException` if a blocked deep link should lead to a safe landing page:

```dart
onException: (context, state, router) {
  if (state.error is BlockedInitialNavigationException) {
    router.go('/');
    return;
  }
  router.go('/error');
},
```

Keep the guard fast enough for navigation, make asynchronous services cancellable where possible, and test both a normal tap and a browser refresh. The key checks are:

- Return `Allow()` for routes that do not need the guard.
- Use `Block.stop()` when staying on the current route is the intended result.
- Use `Block.then` only for a deliberate follow-up navigation.
- Treat initial blocked deep links separately from ordinary blocked transitions.

References: [OnEnter API](https://pub.dev/documentation/go_router/latest/go_router/OnEnter.html), [RoutingConfig API](https://pub.dev/documentation/go_router/latest/go_router/RoutingConfig-class.html), [go_router configuration](https://pub.dev/documentation/go_router/latest/topics/Configuration-topic.html).
