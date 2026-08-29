---
layout: post
title: "go_router fullPath and topRoute - Build Reliable Flutter Route Analytics"
description: "Learn how go_router fullPath and topRoute separate route templates from nested navigation details for stable Flutter analytics and debugging."
date: 2026-08-30
tags: [go_router, navigation, analytics, debugging, Flutter]
comments: true
share: true
---

![go_router fullPath and topRoute for Flutter route analytics](/assets/images/go-router-stateful-shell-route.png)

`go_router`'s `fullPath` and `topRoute` solve a problem that is easy to hide in Flutter analytics: the URL a user opened is not always the route identity you want to report. `/orders/42/items/7` contains useful detail, but a dashboard usually needs to know that the user is on the `/orders/:orderId/items/:itemId` screen, inside the `orders` top-level route. `GoRouterState` now exposes both pieces directly, so route observers and error logs do not need to parse URL strings by hand.

## Why `uri`, `matchedLocation`, and `fullPath` are different

Consider this route tree:

```dart
GoRoute(
  name: 'orders',
  path: '/orders/:orderId',
  routes: [
    GoRoute(
      name: 'item',
      path: 'items/:itemId',
      builder: (context, state) => OrderItemPage(
        orderId: state.pathParameters['orderId']!,
        itemId: state.pathParameters['itemId']!,
      ),
    ),
  ],
),
```

For `/orders/42/items/7?source=push#reviews`, the state values have different jobs:

| Property | Example | Best use |
|---|---|---|
| `uri` | `/orders/42/items/7?source=push#reviews` | The complete location the user opened |
| `matchedLocation` | `/orders/42/items/7` | The matched URL without query and fragment details |
| `fullPath` | `/orders/:orderId/items/:itemId` | A stable route template for grouping |
| `topRoute` | `GoRoute(name: 'orders')` | The first top-level route in the match |
| `pathParameters` | `orderId: 42`, `itemId: 7` | Values needed by the destination |

The mistake I used to make was sending `state.uri.path` as the analytics screen name. That produced a different screen for every order and made reports nearly useless. Sending `fullPath` gives the event a stable identity while keeping the actual URI available as a separate debug field.

## Build a route snapshot at the routing boundary

Keep route extraction in one small function. Pages should receive domain data; analytics code should not be scattered through every `GoRoute.builder`.

```dart
import 'package:go_router/go_router.dart';

class RouteSnapshot {
  const RouteSnapshot({
    required this.screen,
    required this.topLevelRoute,
    required this.path,
    required this.parameters,
  });

  final String screen;
  final String? topLevelRoute;
  final String path;
  final Map<String, String> parameters;
}

RouteSnapshot snapshotFor(GoRouterState state) {
  return RouteSnapshot(
    screen: state.fullPath ?? state.matchedLocation,
    topLevelRoute: state.topRoute?.name,
    path: state.uri.path,
    parameters: Map<String, String>.unmodifiable(state.pathParameters),
  );
}
```

The fallback matters. `fullPath` is nullable, and an error state or an unusual route configuration should not crash the logger while it is trying to describe the original failure. The fallback still gives a useful concrete path.

I keep `screen` and `path` separate on purpose:

```dart
final route = snapshotFor(state);

analytics.logScreenView(
  screenName: route.screen,
  properties: {
    'top_route': route.topLevelRoute ?? 'unknown',
    'path': route.path,
  },
);
```

`screenName` is safe to aggregate because it does not contain the order ID. The `path` field is useful for private debugging, but it may contain identifiers or search terms. I would not send it to a third-party analytics service without checking its privacy policy and the data rules for the app.

## Track changes with a `GoRouter` listener

For a global screen-view event, a router listener is often simpler than putting logging in individual pages. The listener runs after the router's current state changes, so it also sees redirects and browser back navigation.

```dart
class RouteAnalyticsController {
  RouteAnalyticsController(this.router, this.analytics) {
    router.addListener(_handleRouteChanged);
  }

  final GoRouter router;
  final Analytics analytics;
  String? _lastEventKey;

  void _handleRouteChanged() {
    final route = snapshotFor(router.state);
    final eventKey = '${route.screen}|${route.path}';

    if (eventKey == _lastEventKey) return;
    _lastEventKey = eventKey;

    analytics.logScreenView(
      screenName: route.screen,
      properties: {'top_route': route.topLevelRoute ?? 'unknown'},
    );
  }

  void dispose() {
    router.removeListener(_handleRouteChanged);
  }
}
```

The exact source of the current state depends on the `go_router` version and how the application owns its router. In an app where the router is provided through `MaterialApp.router`, I prefer exposing the `GoRouter` instance from the composition root and calling `snapshotFor(router.state)` there. The important part is the de-duplication key: redirects and rebuilds can produce repeated notifications, and an analytics event should represent a meaningful route change rather than every widget rebuild.

For a `StatefulShellRoute`, `topRoute` is especially useful. A branch can contain `/orders`, `/orders/42`, and `/orders/42/items/7`, while the shell keeps multiple navigators alive. Use `topRoute?.name` to group events by the top-level branch, and `fullPath` to distinguish the actual nested destination. Do not infer the active tab by checking whether a string starts with `/orders`; the route tree is the source of truth.

## Use `topRoute` for route-aware policies

The same distinction helps with lightweight policies such as breadcrumbs or debug labels:

```dart
String navigationArea(GoRouterState state) {
  switch (state.topRoute?.name) {
    case 'orders':
      return 'Orders';
    case 'account':
      return 'Account';
    default:
      return 'Other';
  }
}
```

This is different from using `state.name`. `name` describes the route associated with the current state, which may be the deeply nested `item` route. `topRoute` answers the higher-level question: which root branch owns this match? That makes it a better fit for shell navigation, section-level permissions, and coarse analytics.

## Traps worth testing

| Situation | What to verify |
|---|---|
| `/orders/42` → `/orders/43` | `fullPath` stays stable while parameters change |
| `/orders/42/items/7` | `topRoute` still identifies the orders branch |
| Query changes | `uri` changes, but the screen template does not |
| Redirect to sign-in | Analytics records the final visible route, not only the requested URL |
| Browser refresh | The same deep link produces the same `fullPath` |
| Unknown or error route | Logging uses the fallback and does not throw another exception |

Do not put `fullPath` into a URL or use it to load data. It is a route pattern, not the concrete location. Use `pathParameters` for IDs and `uri.queryParameters` for serializable view state. Also avoid treating `topRoute` as a replacement for an explicit navigation model; if two sections intentionally share a top-level route, give analytics a separate product-level event name.

The practical split is simple: use `uri` when you need the exact deep link, `fullPath` when you need a stable route template, and `topRoute` when you need the owning top-level destination. Keeping those meanings separate makes Flutter navigation analytics more useful and prevents nested `go_router` routes from turning into a pile of hand-written string checks.

Reference: [GoRouterState API](https://pub.dev/documentation/go_router/latest/go_router/GoRouterState-class.html).
