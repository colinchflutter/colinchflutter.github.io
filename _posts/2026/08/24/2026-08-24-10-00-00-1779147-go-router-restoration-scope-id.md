---
layout: post
title: "go_router restorationScopeId - Restore Flutter Navigation After App Restart"
description: "Learn how go_router restorationScopeId restores Flutter navigation state, including nested routes, browser behavior, and common restoration traps."
date: 2026-08-24
tags: [go_router, navigation, state_management, Android, iOS]
comments: true
share: true
---
![go_router restorationScopeId restoring Flutter navigation state after an app restart](/assets/images/go-router-stateful-shell-route.png)

`go_router` can restore the last visible Flutter route after the operating system kills and recreates the app, but only when `restorationScopeId` is configured and the route tree is stable. It gives the router a persistent restoration bucket instead of treating every launch as a new navigation session.

## The problem: a route is not the same as app state

Suppose a user opens `/orders/42`, backgrounds the app, and Android removes its process. A normal router often starts at `/` on launch. These settings solve different problems:

| Setting | Restores | Does not restore |
| --- | --- | --- |
| `restorationScopeId` | Navigation stack and locations | API data or tokens |
| Widget `restorationId` | Widget state | Its containing route |
| `extraCodec` | Serializable `extra` values | Live object graphs |

The router can remember `/orders/42`; it cannot recreate an in-memory `Order` or refresh an expired session.

## Configure the router

Give the router a stable scope and pass a rebuildable ID through the path.

```dart
final router = GoRouter(
  restorationScopeId: 'main-router',
  routes: [
    GoRoute(
      path: '/home',
      builder: (context, state) => const HomeScreen(),
      routes: [
        GoRoute(
          path: 'orders/:orderId',
          builder: (context, state) => OrderScreen(
            orderId: state.pathParameters['orderId']!,
          ),
        ),
      ],
    ),
  ],
);

MaterialApp.router(
  routerConfig: router,
);
```

The string value is not special; its stability is. Changing it to `router-v2` intentionally discards the old bucket, which is useful after a breaking route migration.

## Make restored routes safe

Restoration can happen before remote data is ready. A detail page should accept an ID and load the current record, rather than expecting a saved object.

```dart
class OrderScreen extends StatelessWidget {
  const OrderScreen({required this.orderId, super.key});

  final String orderId;

  @override
  Widget build(BuildContext context) {
    return FutureBuilder<Order>(
      future: repository.fetchOrder(orderId),
      builder: (context, snapshot) {
        if (!snapshot.hasData) return const OrderLoadingView();
        return OrderDetails(order: snapshot.data!);
      },
    );
  }
}
```

If `/orders/42` was deleted, the screen can show not-found or redirect to `/orders` instead of deserializing stale memory.

## Traps I found during testing

Hot restart is not process death, so it does not prove restoration works. Test a deep route by backgrounding the app, stopping its process with device tools, and launching it again.

Keep auth redirects active: a restored `/account` route may appear before session refresh finishes. Restoration recovers navigation; it must not bypass access control.

On Flutter Web, the browser URL is already a source of truth. Use `restorationScopeId` mainly for mobile process recreation, and keep paths backward-compatible so old buckets do not point at missing routes.

## Quick checklist

- Use a stable `restorationScopeId`.
- Pass IDs and query values, not live objects.
- Test process recreation, auth redirects, missing records, and route migrations.

Let `go_router` restore where the user was, then let the screen rebuild the data for that location.
