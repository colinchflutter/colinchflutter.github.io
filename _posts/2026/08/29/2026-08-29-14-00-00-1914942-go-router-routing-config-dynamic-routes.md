---
layout: post
title: "go_router routingConfig - Update Flutter Routes Without Recreating the Router"
description: "Learn how go_router routingConfig updates Flutter routes at runtime while preserving navigation state, auth flows, and deep-link behavior."
date: 2026-08-29
tags: [go_router, navigation, state_management, testing, Flutter]
comments: true
share: true
---

![go_router routingConfig updating a Flutter route tree without replacing navigation state](/assets/images/go-router-stateful-shell-route.png)

`GoRouter.routingConfig` lets a Flutter app replace its route definition at runtime without constructing a new router. That matters when the available routes depend on a remote feature flag, an account plan, or an authentication state. The important boundary is the `ValueNotifier<RoutingConfig>`: update that object, and go_router re-evaluates the route tree while keeping the router, observers, and platform integration alive.

## Why recreating GoRouter caused trouble

I initially rebuilt `GoRouter` whenever the signed-in user changed. It looked straightforward, but the replacement router had a new Navigator state. A deep link could be reprocessed, an in-progress stack could disappear, and analytics observers had to be attached again.

| Requirement | Better boundary | Common mistake |
| --- | --- | --- |
| Change which routes exist | `ValueNotifier<RoutingConfig>` | Recreate `GoRouter` in `build` |
| Change access to an existing route | `redirect` or `onEnter` | Remove the route during every auth change |
| Preserve a tab's Navigator stack | Stable shell and branch keys | Rebuild the whole shell |
| Apply a new route list | `routingConfig.value = ...` | Mutate the existing route list in place |

The route configuration should be treated as a new immutable snapshot. The notifier publishes that snapshot; it should not become a mutable global list that several widgets edit independently.

## Build a router around a routing configuration

The following example changes the route tree when an account becomes an admin. The router instance is created once, while the `RoutingConfig` value is replaced when the session changes.

```dart
final routingConfig = ValueNotifier<RoutingConfig>(
  RoutingConfig(
    routes: [
      GoRoute(
        path: '/',
        builder: (context, state) => const HomePage(),
      ),
      GoRoute(
        path: '/login',
        builder: (context, state) => const LoginPage(),
      ),
    ],
    redirect: (context, state) => sessionRedirect(session, state),
  ),
);

final router = GoRouter.routingConfig(
  routingConfig: routingConfig,
  initialLocation: '/',
  errorBuilder: (context, state) => RouteErrorPage(error: state.error),
);
```

The app still uses the normal router configuration entry point:

```dart
MaterialApp.router(routerConfig: router)
```

When the session changes, publish a complete route snapshot. Keeping the public routes in one function makes it harder to accidentally omit `/login` while the user is signed out.

```dart
void applyRoutes(Session session) {
  final routes = <RouteBase>[
    GoRoute(
      path: '/',
      builder: (context, state) => const HomePage(),
    ),
    if (session.isSignedIn)
      GoRoute(
        path: '/account',
        builder: (context, state) => const AccountPage(),
      ),
    if (session.isAdmin)
      GoRoute(
        path: '/admin',
        builder: (context, state) => const AdminPage(),
      ),
    if (!session.isSignedIn)
      GoRoute(
        path: '/login',
        builder: (context, state) => const LoginPage(),
      ),
  ];

  routingConfig.value = RoutingConfig(
    routes: routes,
    redirect: (context, state) => sessionRedirect(session, state),
  );
}
```

This is a route availability mechanism, not an authorization system. A user can already be on `/admin` when a session expires. The updated configuration may produce a routing error or remove the current match, depending on the transition. I still keep a small global redirect for session safety:

```dart
String? sessionRedirect(Session session, GoRouterState state) {
  final isPublic = state.matchedLocation == '/' ||
      state.matchedLocation == '/login';

  if (!session.isSignedIn && !isPublic) {
    return Uri(path: '/login', queryParameters: {
      'from': state.uri.toString(),
    }).toString();
  }
  return null;
}
```

The redirect and the route snapshot serve different jobs. The snapshot describes what the current application is capable of matching. The redirect protects an existing location and can preserve a validated return path.

## The state-preservation trap

`routingConfig` keeps the `GoRouter` object stable, but it does not promise that every page survives a route-tree replacement. A route removed from the new configuration cannot keep its page. A stateful shell also needs stable `GlobalKey<NavigatorState>` instances and consistent branch structure.

I use this rule when feature flags change:

| Route change | Expected result |
| --- | --- |
| Add `/reports` while on `/home` | Current home page remains available |
| Remove `/reports` while on `/home` | No visible disruption |
| Remove the route currently displayed | Redirect to a safe public location |
| Reorder shell branches | Treat as a migration; tab state may no longer map correctly |

Do not use dynamic route configuration as a shortcut for every UI permission. If the route should exist for deep links but display an upgrade screen, keep the route and decide inside `redirect`, `onEnter`, or the page itself. Removing it makes browser refreshes and bookmarked URLs harder to explain.

## Test the update, not just the initial tree

The useful test starts on a stable route, publishes a new configuration, and verifies both the newly available path and the current location. I also test the failure case where the active route is removed.

```dart
testWidgets('publishes an admin route without replacing the router',
    (tester) async {
  applyRoutes(const Session.signedInAdmin());
  await tester.pumpWidget(MaterialApp.router(routerConfig: router));

  router.go('/admin');
  await tester.pumpAndSettle();

  expect(find.byType(AdminPage), findsOneWidget);
  expect(router.routeInformationProvider.value.uri.path, '/admin');
});
```

Avoid asserting only that `AdminPage` appears. A route update can render the right widget while breaking browser history or losing a shell branch's stack. Include direct navigation, refresh restoration on Web, and a session downgrade while an account page is visible.

`GoRouter.routingConfig` is most useful when the route tree genuinely changes at runtime. Keep the router instance and navigator keys stable, publish complete configuration snapshots, and use redirects or entry guards for access decisions. That separation keeps feature flags and authentication changes from turning navigation state into disposable widget state.

References: [GoRouter.routingConfig API](https://pub.dev/documentation/go_router/latest/go_router/GoRouter/GoRouter.routingConfig.html), [RoutingConfig API](https://pub.dev/documentation/go_router/latest/go_router/RoutingConfig-class.html), and [go_router configuration topic](https://pub.dev/documentation/go_router/latest/topics/Configuration-topic.html).
