---
layout: post
title: "go_router GoRouterState.path - Identify Flutter Route Templates Safely"
description: "Learn how GoRouterState.path exposes Flutter route templates, when to use it instead of uri or matchedLocation, and how to avoid brittle URL parsing."
date: 2026-09-11
tags: [go_router, navigation, testing, Web]
comments: true
share: true
---

![go_router exposing a Flutter route template for analytics and navigation policy](/assets/images/go-router-stateful-shell-route.png)

`GoRouterState.path` gives a Flutter page the route template that matched it, such as `/orders/:orderId`, rather than the concrete URL `/orders/42`. That distinction makes it useful for route analytics, breadcrumbs, and policy decisions that should apply to a screen family. It is not a replacement for `GoRouterState.uri`: use `uri` when the actual ID, query, or fragment matters.

## The three strings that look similar

I initially used `state.uri.path` everywhere because it looked like the most direct answer. That worked for `/orders/42`, but it made analytics treat every order as a different screen. Splitting the string or replacing IDs manually also broke as soon as a nested route or encoded value appeared.

The relevant values describe different layers of the same match:

| Value | Example | Best use |
| --- | --- | --- |
| `state.path` | `/orders/:orderId` | Route-template policies and grouped analytics |
| `state.matchedLocation` | `/orders/42` | Concrete matched path without query or fragment |
| `state.uri` | `/orders/42?tab=history#notes` | Complete URL state |
| `state.fullPath` | `/orders/:orderId/items/:itemId` | Full nested route pattern |

`path` belongs to the route associated with the current `GoRouterState`. `fullPath` describes the entire matched pattern. With nested routes, that difference matters.

```dart
final router = GoRouter(
  routes: [
    GoRoute(
      path: '/orders/:orderId',
      builder: (context, state) => const OrderPage(),
      routes: [
        GoRoute(
          path: 'items/:itemId',
          builder: (context, state) => const OrderItemPage(),
        ),
      ],
    ),
  ],
);
```

For `/orders/42/items/7`, a child route can see values similar to these:

```dart
final routeTemplate = state.path;
// items/:itemId

final completeTemplate = state.fullPath;
// /orders/:orderId/items/:itemId

final actualUrl = state.uri.toString();
// /orders/42/items/7
```

The exact value of `path` is local to the route match, so I use `fullPath` when the metric needs to identify the complete screen. I use `path` when a feature owns a single route level and does not need to know its parent.

## Group screen analytics by route template

A page-view event should usually answer “which screen did the user see?” rather than “which URL string did the user visit?” Sending `state.uri.path` as the screen name produces one event name for every product, article, or order ID.

The route template gives analytics a stable dimension:

```dart
class RouteAnalytics {
  void record(GoRouterState state) {
    final screen = state.fullPath ?? state.path ?? 'unknown';

    analytics.logEvent(
      name: 'screen_view',
      parameters: {
        'screen': screen,
        'route_name': state.name ?? 'unnamed',
        'has_query': state.uri.hasQuery,
      },
    );
  }
}
```

The ID can still be recorded separately when it is safe and useful:

```dart
final orderId = state.pathParameters['orderId'];

analytics.logEvent(
  name: 'order_screen_view',
  parameters: {
    'screen': state.fullPath ?? state.path ?? 'unknown',
    'order_id': orderId,
  },
);
```

In a real analytics pipeline, I would avoid sending raw user IDs or sensitive values without a data policy. The important design is the separation: the route template identifies the screen, while path parameters are optional event context.

## Use `path` for route-aware policies

Some UI rules apply to a class of routes. For example, a desktop app may show a secondary inspector on every editor route, while a mobile app may hide it elsewhere. Comparing `state.uri.path` to `/editor/42` is fragile because the ID changes.

```dart
bool showsInspector(GoRouterState state) {
  final template = state.fullPath ?? state.path;

  return template == '/editor/:documentId' ||
      template == '/editor/:documentId/preview';
}
```

If the route tree has more nesting than this policy understands, a route name or metadata value can be clearer than comparing templates. `path` is useful when the route itself is the source of truth; it should not become a second, undocumented route registry.

## Breadcrumbs need a different value

`path` is a developer-facing pattern, so it is a poor label for a user-facing breadcrumb. Rendering `/orders/:orderId` would expose implementation details. Build the label from the route template, then resolve the dynamic value separately:

```dart
String breadcrumbLabel(GoRouterState state) {
  switch (state.fullPath) {
    case '/orders/:orderId':
      final id = state.pathParameters['orderId'];
      return id == null ? 'Order' : 'Order $id';
    case '/orders/:orderId/items/:itemId':
      return 'Order item';
    default:
      return 'Page';
  }
}
```

This is intentionally small. For a large application, route metadata or typed route data can keep labels beside route definitions. The trap is not the switch itself; the trap is using a concrete URL as if it were a stable route identity.

## Avoid reading `path` from an unrelated context

`GoRouterState.path` is available in the `GoRoute` builder, redirect, and other code that has a route state. A reusable widget can read the state with `GoRouterState.of(context)`, but that widget must be below the route in the tree:

```dart
class RouteDebugBadge extends StatelessWidget {
  const RouteDebugBadge({super.key});

  @override
  Widget build(BuildContext context) {
    final state = GoRouterState.of(context);

    return Text(state.fullPath ?? state.path ?? 'unknown route');
  }
}
```

Calling this widget in a standalone widget test without a router ancestor produces a lookup failure. Either pump the real `MaterialApp.router`, or pass a small route model into the widget when route awareness is not its responsibility.

```dart
testWidgets('shows the route template', (tester) async {
  final router = GoRouter(
    initialLocation: '/orders/42',
    routes: [
      GoRoute(
        path: '/orders/:orderId',
        builder: (context, state) => const RouteDebugBadge(),
      ),
    ],
  );

  await tester.pumpWidget(
    MaterialApp.router(routerConfig: router),
  );

  expect(find.text('/orders/:orderId'), findsOneWidget);
});
```

## Practical rules

- Use `state.path` for the current route's template.
- Use `state.fullPath` when nested parents should be part of the identity.
- Use `state.uri` for query parameters, fragments, and the concrete URL.
- Use `state.matchedLocation` when the matched concrete path should exclude query data.
- Keep route templates for diagnostics and policies; turn them into friendly labels before rendering them to users.

`GoRouterState.path` is a small property, but it prevents a common category error: treating a route pattern, a matched URL, and complete URL state as the same thing. Once those values have separate jobs, analytics stays grouped, policies survive dynamic IDs, and deep-link behavior is easier to test.
