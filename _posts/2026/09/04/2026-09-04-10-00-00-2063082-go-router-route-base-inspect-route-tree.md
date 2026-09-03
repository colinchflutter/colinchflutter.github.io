---
layout: post
title: "go_router RouteBase - Inspect a Flutter Route Tree Without Parsing URLs"
description: "Learn how to use go_router RouteBase to inspect Flutter GoRoute and ShellRoute trees for breadcrumbs, diagnostics, and route policies."
date: 2026-09-04
tags: [go_router, navigation, Flutter, performance]
comments: true
share: true
---

![go_router RouteBase entries forming a Flutter route tree](/assets/images/go-router-stateful-shell-route.png)

`go_router`'s `RouteBase` gives a Flutter app a useful view of its declared route tree. Instead of parsing URL strings or maintaining a second list of routes, I can walk the same `GoRoute` and `ShellRoute` objects used by `GoRouter` and build breadcrumbs, diagnostics, or navigation policies from one source.

## Why a route tree is different from the current location

I first tried to create a navigation menu by splitting `state.uri.path` on `/`. That worked until a shell route introduced a tab navigator. The URL showed `/account/orders`, but the UI also needed to know that `account` was a shell boundary and that `orders` was a leaf page.

The two pieces of information answer different questions:

| Value | Describes | Best use |
| --- | --- | --- |
| `state.uri` | The current URL and query data | Deep links and filters |
| `state.matchedLocation` | The matched URL string | URL-based analytics |
| `RouteBase` tree | The declared route structure | Menus, audits, and policies |

`RouteBase` is not a replacement for `GoRouterState`. It is useful before navigation happens, when the application needs to understand its configured routes rather than the page currently being rendered.

## Keep the route list as a reusable source

The small design decision that matters is storing the routes in a named variable. Passing an inline list directly to `GoRouter` makes the route tree harder to inspect elsewhere.

Here is a route configuration containing both a shell and ordinary child routes:

```dart
import 'package:flutter/material.dart';
import 'package:go_router/go_router.dart';

final List<RouteBase> appRoutes = <RouteBase>[
  ShellRoute(
    builder: (context, state, child) {
      return AppShell(child: child);
    },
    routes: <RouteBase>[
      GoRoute(
        path: '/home',
        name: 'home',
        builder: (context, state) => const HomeScreen(),
      ),
      GoRoute(
        path: '/orders/:orderId',
        name: 'order',
        builder: (context, state) {
          return OrderScreen(
            orderId: state.pathParameters['orderId']!,
          );
        },
      ),
    ],
  ),
  GoRoute(
    path: '/login',
    name: 'login',
    builder: (context, state) => const LoginScreen(),
  ),
];

final GoRouter router = GoRouter(routes: appRoutes);
```

The list type is deliberately `List<RouteBase>`. Both `GoRoute` and `ShellRoute` are route-base entries, so the type makes nested configuration explicit and lets helper functions accept either kind.

## Walk nested routes for diagnostics

`RouteBase` itself does not provide one universal `path` or `name`; a shell route is a container while a `GoRoute` owns a path. A recursive helper can handle that distinction without relying on URL parsing.

The following function creates readable route records for a debug screen or a test:

```dart
class RouteRecord {
  const RouteRecord({
    required this.kind,
    required this.path,
    required this.name,
    required this.depth,
  });

  final String kind;
  final String path;
  final String? name;
  final int depth;
}

List<RouteRecord> inspectRoutes(
  List<RouteBase> routes, {
  int depth = 0,
}) {
  final records = <RouteRecord>[];

  for (final route in routes) {
    if (route is GoRoute) {
      records.add(RouteRecord(
        kind: 'GoRoute',
        path: route.path,
        name: route.name,
        depth: depth,
      ));
      records.addAll(inspectRoutes(route.routes, depth: depth + 1));
    } else if (route is ShellRoute) {
      records.add(RouteRecord(
        kind: 'ShellRoute',
        path: '<shell>',
        name: null,
        depth: depth,
      ));
      records.addAll(inspectRoutes(route.routes, depth: depth + 1));
    }
  }

  return records;
}
```

The output is intentionally metadata, not widgets. That keeps the inspector safe to call in tests or developer tooling without constructing pages. It also exposes a common mistake: shell routes are structural nodes, so treating every `RouteBase` as a page produces incorrect breadcrumbs.

## Turn the same tree into a route policy

A route audit is more useful when it fails early. For example, named navigation depends on unique route names. A lightweight test can inspect the configured tree and catch duplicate names before a user taps a menu item.

```dart
void assertUniqueRouteNames(List<RouteBase> routes) {
  final names = <String>{};

  for (final route in routes) {
    if (route is GoRoute && route.name != null) {
      if (!names.add(route.name!)) {
        throw StateError('Duplicate go_router name: ${route.name}');
      }
    }

    if (route.routes.isNotEmpty) {
      assertUniqueRouteNames(route.routes);
    }
  }
}
```

Call this from a debug-only startup check or a unit test. Do not run expensive route analysis on every build; the route configuration is normally static, so compute records once and cache them if a diagnostics page needs them repeatedly.

## Traps I hit

`RouteBase` does not tell you which route is currently active. Use `GoRouterState` inside a builder or redirect for that decision. The route tree says what can exist, not what is visible now.

Also, a child `GoRoute` path should be interpreted according to its parent configuration. A string such as `settings` may be relative in a nested route, so concatenating raw `route.path` values can create a misleading full path. For a production breadcrumb, combine the declared structure with the active match information rather than assuming every path is absolute.

Finally, keep inspection helpers read-only. Mutating route lists after the router is created makes debugging much harder and may leave the router's internal configuration out of sync.

## Short checklist

- Store the shared configuration as `List<RouteBase>`.
- Treat `GoRoute` as a page node and `ShellRoute` as a container node.
- Walk `route.routes` recursively for audits and developer tooling.
- Use `GoRouterState` when the question is the active location.
- Cache static inspection results instead of rebuilding them with every widget.
