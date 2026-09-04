---
layout: post
title: "go_router GoRouterDelegate.currentConfiguration - Observe Flutter Navigation Safely"
description: "Learn how to use go_router GoRouterDelegate.currentConfiguration to observe Flutter route changes without rebuilding the router or parsing URLs manually."
date: 2026-09-04
tags: [go_router, navigation, Flutter, performance]
comments: true
share: true
---

![go_router GoRouterDelegate observing Flutter navigation changes](/assets/images/go-router-observers-navigation-scope.png)

The safest way to observe `go_router` navigation is to listen to the router delegate and read its `currentConfiguration`, rather than parsing URLs or rebuilding `GoRouter` whenever a screen changes. The value is a `RouteMatchList`, so it represents the route configuration that the Router system is currently using.

_The diagram shows the navigation observer boundary to keep in mind: listen to the router once, then let widgets consume a small piece of derived state._

## Why `router.location` is not enough

I originally used the current URL to decide whether a compact screen should show a back button. That looked simple, but it mixed three separate questions:

| Question | Better source |
| --- | --- |
| What URL is visible? | `GoRouterState.uri` or the route information provider |
| Which matches are active? | `GoRouterDelegate.currentConfiguration` |
| Which screen is on top? | The last relevant match in the `RouteMatchList` |

The difference matters with nested routes and shell navigation. A location such as `/settings/profile` does not tell a UI component whether the profile page is inside a `ShellRoute`, whether an imperative page was pushed above it, or how many route matches are active. `currentConfiguration` is useful when that structure—not just the URL—is the input to a decision.

## Reading the active configuration

`GoRouter` exposes its `RouterDelegate` through `routerDelegate`. The delegate is a `Listenable`, so a small `ChangeNotifier` can convert route changes into a focused value for the widget tree.

Here is a compact notifier that tracks the active match list and removes its listener correctly:

```dart
import 'package:flutter/foundation.dart';
import 'package:go_router/go_router.dart';
import 'package:provider/provider.dart';

class RouteConfigurationNotifier extends ChangeNotifier {
  RouteConfigurationNotifier(this.router) {
    router.routerDelegate.addListener(_handleRouteChanged);
    _configuration = router.routerDelegate.currentConfiguration;
  }

  final GoRouter router;
  late RouteMatchList _configuration;

  RouteMatchList get configuration => _configuration;

  void _handleRouteChanged() {
    final next = router.routerDelegate.currentConfiguration;
    if (next == _configuration) return;

    _configuration = next;
    notifyListeners();
  }

  @override
  void dispose() {
    router.routerDelegate.removeListener(_handleRouteChanged);
    super.dispose();
  }
}
```

The equality check is deliberate. Router notifications can happen for framework reasons that do not change the route data a consumer needs. Avoiding an unnecessary `notifyListeners()` prevents every dependent widget from rebuilding for those events.

## Exposing only the value a widget needs

The notifier should usually live near the router rather than being created inside every page. With `provider`, for example, it can be provided beside `MaterialApp.router`:

```dart
final router = GoRouter(
  routes: [
    GoRoute(path: '/', builder: (_, __) => const HomeScreen()),
    GoRoute(path: '/settings', builder: (_, __) => const SettingsScreen()),
  ],
);

class App extends StatefulWidget {
  const App({super.key});

  @override
  State<App> createState() => _AppState();
}

class _AppState extends State<App> {
  late final RouteConfigurationNotifier routeState;

  @override
  void initState() {
    super.initState();
    routeState = RouteConfigurationNotifier(router);
  }

  @override
  void dispose() {
    routeState.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return ChangeNotifierProvider.value(
      value: routeState,
      child: MaterialApp.router(routerConfig: router),
    );
  }
}
```

For a project without `provider`, the same notifier works with `ValueListenableBuilder` after adapting it to expose a `ValueNotifier<RouteMatchList>`. The important ownership rule stays the same: create one observer for one router and dispose it with that router's lifecycle.

## Turning matches into route-aware UI

Do not make every widget understand the internals of `RouteMatchList`. Convert it at the boundary into a stable view model, such as `canPop`, `isSettingsArea`, or an analytics name. This keeps navigation API details out of presentation code.

```dart
class RouteChrome extends StatelessWidget {
  const RouteChrome({super.key});

  @override
  Widget build(BuildContext context) {
    final matches = context.watch<RouteConfigurationNotifier>().configuration;
    final hasNestedPage = matches.matches.length > 1;

    return AppBar(
      leading: hasNestedPage ? const BackButton() : null,
      title: Text(_titleFor(matches)),
    );
  }

  String _titleFor(RouteMatchList configuration) {
    final location = configuration.uri.path;
    if (location.startsWith('/settings')) return 'Settings';
    return 'Home';
  }
}
```

The `matches.length` check is only an example. Shell routes, redirects, and `ImperativeRouteMatch` can make a simple count unsuitable for a production back-button policy. If the decision is “can the current navigator pop?”, ask that navigator directly. Use the match list when the decision is about declarative route structure.

## Common traps

The first trap is recreating the router in `build`. That resets listeners and can make an observer appear to miss transitions. Keep `GoRouter` and the notifier in stable state.

The second trap is assuming every notification means a new page. `RouterDelegate` participates in Flutter's Router protocol, so listen for changes but compare the configuration before publishing derived state.

The third trap is treating `currentConfiguration` as an analytics event stream. It describes current state; it does not tell you why the state changed. For event-level telemetry, add a navigation observer or instrument the application commands as well. Use this delegate listener for snapshots such as route-aware chrome, diagnostics, and policy checks.

## Short takeaway

`GoRouterDelegate.currentConfiguration` is a practical boundary between Flutter's Router machinery and application UI. Listen once, compare the new `RouteMatchList`, derive a small view model, and dispose the listener with the router owner. That gives nested and shell routes a structural source of truth without maintaining a second navigation stack.

References: [GoRouter API](https://pub.dev/documentation/go_router/latest/go_router/GoRouter-class.html), [GoRouterDelegate API](https://pub.dev/documentation/go_router/latest/go_router/GoRouterDelegate-class.html), and [Flutter RouterDelegate](https://api.flutter.dev/flutter/widgets/RouterDelegate-class.html).
