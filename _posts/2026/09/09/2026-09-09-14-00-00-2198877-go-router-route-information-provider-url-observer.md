---
layout: post
title: "go_router routeInformationProvider - Observe Flutter URL Changes Safely"
description: "Learn how go_router routeInformationProvider exposes Flutter URL changes, when to use it instead of GoRouterState.uri, and how to avoid feedback loops."
date: 2026-09-09
tags: [go_router, navigation, Web, testing]
comments: true
share: true
---

![go_router routeInformationProvider observing Flutter URL changes](/assets/images/go-router-stateful-shell-route.png)

The key detail is the separation between a central URL observer and the nested navigators that render each branch.

`go_router`'s `routeInformationProvider` is the right boundary when an app-level service needs to observe the URL produced by the router. It is useful for browser analytics, a document-title bridge, or a debug panel. It is not a replacement for `GoRouterState.uri` inside a page, and treating it like one usually creates unnecessary listeners and URL feedback loops.

## Choose the right source

| Need | Best source | Why |
| --- | --- | --- |
| Read parameters while building a page | `GoRouterState.uri` | Already scoped to the matched route |
| Observe every URL change centrally | `routeInformationProvider` | Emits router-level `RouteInformation` |
| Inspect the active page structure | `routerDelegate.currentConfiguration` | Contains route matches, not only a string |

The provider exposes a `ValueListenable<RouteInformation>`. A small widget can subscribe without parsing button callbacks throughout the app:

```dart
class UrlObserver extends StatefulWidget {
  const UrlObserver({required this.router, super.key});

  final GoRouter router;

  @override
  State<UrlObserver> createState() => _UrlObserverState();
}

class _UrlObserverState extends State<UrlObserver> {
  @override
  void initState() {
    super.initState();
    widget.router.routeInformationProvider.addListener(_onUrlChanged);
  }

  void _onUrlChanged() {
    final information = widget.router.routeInformationProvider.value;
    final uri = information.uri;
    debugPrint('route changed: ${uri.path}?${uri.query}');
  }

  @override
  void dispose() {
    widget.router.routeInformationProvider.removeListener(_onUrlChanged);
    super.dispose();
  }

  @override
  Widget build(BuildContext context) => const SizedBox.shrink();
}
```

The lifecycle detail matters: remove the listener from the same provider before disposal. If the router can be replaced, resubscribe in `didUpdateWidget`; otherwise the observer keeps listening to the old router.

## Do not write back from the observer

The provider reports navigation. It should not normally call `go()` or `push()` in the same callback. A URL observer that redirects on every notification can produce a loop, duplicate history entries, or a browser address bar that fights the user's back button.

Keep policy in `redirect` or `onEnter`, and keep observation side-effect-light:

```dart
void _onUrlChanged() {
  final uri = widget.router.routeInformationProvider.value.uri;
  analytics.trackScreen(uri.path, query: uri.queryParameters);
}
```

Query parameters are available through the `Uri`; do not split `RouteInformation.location` manually. On Web, test a direct link and browser back/forward. On mobile, test both `go()` and system back because the timing can differ.

`routeInformationProvider` is a central observation point, not a page-data API. Use it for cross-cutting URL effects, keep route decisions in the router configuration, and dispose listeners explicitly. That separation makes browser history and nested `StatefulShellRoute` navigation easier to reason about.
