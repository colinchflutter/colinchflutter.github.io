---
layout: post
title: "go_router RouteMatchList - Inspect the Active Flutter Route Stack Safely"
description: "Use go_router RouteMatchList to inspect Flutter's active route stack, build route-aware UI, and debug go, push, and ShellRoute behavior."
date: 2026-09-03
tags: [go_router, navigation, debugging, Flutter, performance]
comments: true
share: true
---

![go_router route matches forming an active Flutter navigation stack](/assets/images/go-router-stateful-shell-route.png)

`go_router` already knows which routes are active, but many Flutter apps still try to reconstruct that information by parsing `router.location` or watching every button that calls `go()` and `push()`. That approach breaks as soon as nested routes, `ShellRoute`, or browser back navigation enter the picture. `RouteMatchList` is the better source when you need to inspect the active route stack.

## What RouteMatchList represents

The router's current configuration is a `RouteMatchList`. It is the result of matching the current URI against the route tree. Unlike a single location string, it preserves the matched route structure and can contain nested or shell matches.

The useful pieces are small enough to map directly to common tasks:

| Property | Useful for |
| --- | --- |
| `uri` | The canonical URL, including query and fragment |
| `matches` | Inspecting the active match objects |
| `last` / `lastOrNull` | Finding the leaf route currently on screen |
| `routes` | Reading the matched `RouteBase` definitions |
| `pathParameters` | Logging parameters after URI decoding |
| `isError` | Detecting a configuration that will show an error page |

The important distinction is that `matches` describes the route matches, not every `Page` currently stored in a Navigator. Imperatively pushed pages are represented differently, and a `ShellRoute` can add a nested match tree. Treat this as routing state, not as a replacement for `Navigator` history.

## Read the current match list

The following widget reads the active configuration without maintaining a second route state in the application:

```dart
class RouteStackPanel extends StatelessWidget {
  const RouteStackPanel({super.key});

  @override
  Widget build(BuildContext context) {
    final router = GoRouter.of(context);
    final configuration = router.routerDelegate.currentConfiguration;
    final matches = configuration.matches;

    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        Text('URL: ${configuration.uri}'),
        for (final match in matches)
          Text(match.route.runtimeType.toString()),
      ],
    );
  }
}
```

`currentConfiguration` is a snapshot. The widget must rebuild when the router changes, so a permanent panel should be placed under a router-aware rebuild mechanism or rebuilt by the screen that owns the navigation shell. Reading the value once in `initState` is a common failed assumption: it captures the initial route and never becomes a live view.

For a small diagnostic overlay, `GoRouter` can be supplied as a `Listenable` to a `ListenableBuilder`:

```dart
class RouteDiagnostics extends StatelessWidget {
  const RouteDiagnostics({required this.router, super.key});

  final GoRouter router;

  @override
  Widget build(BuildContext context) {
    return ListenableBuilder(
      listenable: router,
      builder: (context, child) {
        final configuration = router.routerDelegate.currentConfiguration;
        final leaf = configuration.lastOrNull;

        return Text(
          'leaf=${leaf?.route.path ?? '-'}\n'
          'uri=${configuration.uri}',
        );
      },
    );
  }
}
```

This is particularly useful during development. It exposes whether a tap performed `go()` or `push()` as expected, whether a redirect settled on the intended route, and whether a query change actually produced a new URI.

## Flatten nested matches for breadcrumbs

With a simple `GoRoute` tree, `configuration.matches` may be enough. With `ShellRoute`, however, a match can contain another match list. A breadcrumb builder should therefore visit the match tree instead of assuming every item is a plain `RouteMatch`.

```dart
List<String> activeRoutePaths(RouteMatchList configuration) {
  final paths = <String>[];

  configuration.visitRouteMatches((match) {
    if (match is RouteMatch) {
      paths.add(match.route.path);
    }
    return true;
  });

  return paths;
}
```

The visitor callback's return value controls whether nested matches are visited. Keep the traversal separate from the widget so it can be tested with representative shell and child routes. A route label should normally come from explicit metadata or a route-name map; a raw path such as `:projectId` is a template, not a user-facing breadcrumb.

## `go()` and `push()` are not interchangeable

The match list is also a good way to explain navigation bugs. `context.go('/projects/42')` replaces the current declarative match list with the destination. `context.push('/projects/42')` adds an imperative match on top of the current configuration. Their URLs can look similar while back-button behavior differs.

| Operation | Match-stack expectation | Typical use |
| --- | --- | --- |
| `go()` | Replace the route configuration | Tabs, deep links, signed-in destinations |
| `push()` | Add a page above the current route | Details, compose, temporary flows |
| `replace()` | Replace the top pushed match | Editing a pushed step |

Do not use `configuration.matches.length` as a generic “number of screens” metric. A shell branch, a nested route, and an imperative match do not have the same meaning. For analytics, record the URI and leaf route; for UI, derive only the specific state you need.

## Keep inspection out of production decisions

It is tempting to hide a bottom bar whenever `matches.length > 1`, or to authorize a page by checking the last path string. Both rules become fragile with nested navigators and redirects. Route configuration is excellent for diagnostics, breadcrumbs, and route-aware presentation. Authentication and business permissions should remain explicit in `onEnter`, `redirect`, or application state.

A focused test protects the route-inspection contract:

```dart
testWidgets('route diagnostics follows navigation', (tester) async {
  final router = GoRouter(
    initialLocation: '/home',
    routes: [
      GoRoute(path: '/home', builder: (_, __) => const Text('Home')),
      GoRoute(path: '/settings', builder: (_, __) => const Text('Settings')),
    ],
  );

  await tester.pumpWidget(
    MaterialApp.router(routerConfig: router),
  );
  expect(router.routerDelegate.currentConfiguration.uri.path, '/home');

  router.go('/settings');
  await tester.pumpAndSettle();
  expect(router.routerDelegate.currentConfiguration.last.route.path, '/settings');
});
```

`RouteMatchList` is a compact view of what `go_router` matched, not a second navigation framework. Use its URI and leaf match for stable diagnostics, traverse nested matches when building route-aware UI, and avoid treating match counts as page counts. That keeps route inspection useful even when the app grows from a flat route list into shells, redirects, and deep links.

References: [RouteMatchList API](https://pub.dev/documentation/go_router/latest/go_router/RouteMatchList-class.html), [go_router configuration](https://pub.dev/documentation/go_router/latest/topics/Configuration-topic.html), and [go_router navigation](https://pub.dev/documentation/go_router/latest/topics/Navigation-topic.html).
