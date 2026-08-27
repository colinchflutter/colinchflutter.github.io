---
layout: post
title: "go_router observers - Track Flutter Navigation Without Duplicate Analytics"
description: "Learn how go_router observers capture Flutter route events, how shell observers differ from root observers, and how to prevent duplicate screen tracking."
date: 2026-08-27
tags: [go_router, navigation, testing, performance, Flutter]
comments: true
share: true
---

![go_router root and shell NavigatorObserver scopes in Flutter](/assets/images/go-router-observers-navigation-scope.png)

The safest way to track Flutter navigation with `go_router` is to assign each `NavigatorObserver` a clear scope. Put global analytics on `GoRouter.observers`, local tab behavior on a `ShellRoute` observer, and decide explicitly whether shell events should also reach the root observer. Otherwise one screen view can be recorded twice, or a tab transition can disappear from your tracking entirely.

## Why one observer is not enough for every route

`go_router` can build more than one `Navigator`. The root router owns ordinary top-level pages, while `ShellRoute` and `StatefulShellRoute` can create nested navigators. An observer only receives notifications from the navigator to which it is attached.

| Observer location | Sees | Good use | Common mistake |
| --- | --- | --- | --- |
| `GoRouter.observers` | Root navigator events | Global screen analytics | Assuming it sees every shell push |
| `ShellRoute.observers` | That shell's navigator events | Tab-local logging or scroll restoration | Sending the same event to analytics again |
| `notifyRootObserver: true` | Shell events forwarded to root observer | One centralized analytics pipeline | Counting both local and forwarded events |

That scope is easy to miss because the visible page still changes normally. The bug usually appears later in an analytics dashboard: root pages have one event, while tab pages have two or zero.

## Register a root observer

Start with a small observer that records route names but does not assume every `Route` has a non-null name. A `GoRouter` page may be represented by a `Page` whose settings contain a location rather than a useful name.

```dart
import 'package:flutter/material.dart';

class AppNavigationObserver extends NavigatorObserver {
  void _track(Route<dynamic>? route, String action) {
    final settings = route?.settings;
    final name = settings?.name ?? '<unnamed>';

    debugPrint('[navigation] $action $name');
  }

  @override
  void didPush(Route<dynamic> route, Route<dynamic>? previousRoute) {
    super.didPush(route, previousRoute);
    _track(route, 'push');
  }

  @override
  void didPop(Route<dynamic> route, Route<dynamic>? previousRoute) {
    super.didPop(route, previousRoute);
    _track(route, 'pop');
  }

  @override
  void didReplace({Route<dynamic>? newRoute, Route<dynamic>? oldRoute}) {
    super.didReplace(newRoute: newRoute, oldRoute: oldRoute);
    _track(newRoute, 'replace');
  }
}
```

Register one instance on the router. Keeping the instance stable matters if the analytics client stores a reference to it.

```dart
final rootNavigationObserver = AppNavigationObserver();

final router = GoRouter(
  observers: [rootNavigationObserver],
  routes: [
    GoRoute(
      path: '/',
      builder: (context, state) => const HomePage(),
    ),
  ],
);
```

This observer covers routes pushed onto the root navigator. It does not automatically make a nested shell navigator part of the same event stream.

## Add a local observer to a shell

A shell observer is useful when the shell owns behavior that should stay local, such as tracking tab stack depth or debugging a nested back-button flow.

```dart
final tabNavigationObserver = AppNavigationObserver();

final router = GoRouter(
  observers: [rootNavigationObserver],
  routes: [
    ShellRoute(
      observers: [tabNavigationObserver],
      // Forward shell notifications to the root observer only if desired.
      notifyRootObserver: false,
      builder: (context, state, child) {
        return AppScaffold(child: child);
      },
      routes: [
        GoRoute(
          path: '/feed',
          builder: (context, state) => const FeedPage(),
        ),
        GoRoute(
          path: '/settings',
          builder: (context, state) => const SettingsPage(),
        ),
      ],
    ),
  ],
);
```

With `notifyRootObserver: false`, a tab route is reported to `tabNavigationObserver` only. That is a good setup when the local observer has a separate purpose and analytics is emitted by a shell-aware service.

If the root observer is the single analytics pipeline, set `notifyRootObserver: true` and remove analytics emission from the local observer. The local observer can still handle UI-specific work, but it should not call the same `trackScreenView` method.

## Stateful shells need an explicit policy

`StatefulShellRoute` keeps branch navigators alive, so switching from one tab to another is not identical to pushing a new root page. A branch may preserve its previous stack, and an observer attached to that branch can see events that a root observer does not.

I use this policy for a bottom-navigation app:

```text
Root observer       -> global pages and one screen-view pipeline
Shell observer      -> tab-local push/pop diagnostics only
Branch restoration  -> state, not a new analytics event
Tab selection       -> one explicit "tab_selected" event
```

The distinction between a route event and a tab-selection event is important. Restoring a branch should not look like a fresh user visit every time the shell rebuilds.

## Test the event boundary

Do not test only `context.go('/')` and assume the observer setup works. A small recording observer makes the boundary visible.

```dart
class RecordingObserver extends NavigatorObserver {
  final events = <String>[];

  @override
  void didPush(Route<dynamic> route, Route<dynamic>? previousRoute) {
    events.add('push:${route.settings.name}');
  }
}
```

Exercise a root route, a shell child, a browser back action, and a tab switch. Assert which recorder changed and, when forwarding is enabled, assert that the analytics layer deduplicates the forwarded event. A test that checks only the final widget can miss an event-counting regression.

## Traps worth checking before shipping

- `NavigatorObserver` callbacks describe navigator changes, not every URL parsing or redirect evaluation. Use router diagnostics when investigating redirects.
- Do not use `route.settings.name` as the only analytics identity when custom pages produce unnamed routes. Pass a stable route key or normalize the location separately.
- Avoid registering the same observer instance in multiple navigator lists unless receiving both scopes is intentional.
- With a shell, choose either root forwarding or local analytics for a screen view. Using both creates duplicate events.

The useful mental model is simple: `go_router` gives each navigator its own event boundary. Put one global responsibility on the root observer, keep shell-specific behavior local, and treat `notifyRootObserver` as an event-forwarding choice rather than a harmless default. That makes navigation analytics predictable even when the app grows from one stack to persistent tabs and nested flows.

References: [GoRouter API](https://pub.dev/documentation/go_router/latest/go_router/GoRouter-class.html), [ShellRoute API](https://pub.dev/documentation/go_router/latest/go_router/ShellRoute-class.html), and [NavigatorObserver API](https://api.flutter.dev/flutter/widgets/NavigatorObserver-class.html).
