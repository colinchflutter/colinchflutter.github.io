---
layout: post
title: "go_router matchedLocation - Build Reliable Flutter Breadcrumbs and Route Analytics"
description: "Learn how go_router matchedLocation differs from uri and fullPath, then use it for Flutter breadcrumbs, analytics, and route-aware UI."
date: 2026-09-02
tags: [go_router, navigation, FlutterWeb, analytics]
comments: true
share: true
---

![go_router matchedLocation connecting a Flutter route tree to analytics](/assets/images/go-router-stateful-shell-route.png)

`GoRouterState.matchedLocation` is the practical choice when Flutter UI needs the concrete route path that matched, but should ignore query strings and fragments. I use it for selected shell tabs and breadcrumbs; I use `uri` for the complete address and `fullPath` for a route pattern.

## Do not compare the whole URI

Consider a nested route such as `/catalog/item/42?ref=home`. These properties have different jobs:

| Property | Meaning | Example |
|---|---|---|
| `uri` | Complete parsed URL | `/catalog/item/42?ref=home` |
| `matchedLocation` | Concrete matched path | `/catalog/item/42` |
| `fullPath` | Route pattern | `/catalog/item/:itemId` |
| `pathParameters` | Values from placeholders | `itemId: 42` |

Checking `state.uri.toString() == '/catalog'` breaks when a detail route or filter is added. It also makes a query parameter change the selected navigation section, even when the user is still inside Catalog.

## Keep a shell tab selected

The following helper keeps Catalog selected for both `/catalog` and `/catalog/item/42`:

```dart
int selectedSection(GoRouterState state) {
  final location = state.matchedLocation;

  if (location == '/catalog' || location.startsWith('/catalog/')) {
    return 0;
  }
  if (location == '/orders' || location.startsWith('/orders/')) {
    return 1;
  }
  if (location == '/settings' || location.startsWith('/settings/')) {
    return 2;
  }
  return 0;
}
```

The slash after the section name matters. A plain `startsWith('/catalog')` would incorrectly classify `/catalog-settings` as a Catalog child. I initially missed this when a similarly named top-level route was introduced.

Use the result in a `NavigationBar` inside a `ShellRoute` or `StatefulShellRoute`:

```dart
NavigationBar(
  selectedIndex: selectedSection(state),
  onDestinationSelected: (index) {
    const locations = ['/catalog', '/orders', '/settings'];
    context.go(locations[index]);
  },
  destinations: const [
    NavigationDestination(icon: Icon(Icons.grid_view), label: 'Catalog'),
    NavigationDestination(icon: Icon(Icons.receipt_long), label: 'Orders'),
    NavigationDestination(icon: Icon(Icons.settings), label: 'Settings'),
  ],
)
```

## Breadcrumbs and analytics need different values

`matchedLocation` is convenient for simple breadcrumb links because dynamic IDs remain in the concrete path. It is not a good display label, though. `/catalog/item/42` should normally become “Item · Wireless Mouse” using loaded product data, not “42”. For complicated routes, read `state.pathParameters['itemId']` instead of parsing segments manually.

Analytics has the opposite concern. Sending `matchedLocation` as a screen name creates separate reports for item 42 and item 99. Send `fullPath` as the grouped screen name and the ID as a separate dimension:

```dart
analytics.screenView(
  name: state.fullPath ?? 'unknown',
  parameters: {
    'item_id': state.pathParameters['itemId'],
  },
);
```

Keep one event boundary. A `NavigatorObserver` records page-stack events, while router-state tracking records URL changes. Logging both can count one `go()` twice, especially when a route configuration is replaced.

## Quick checks before shipping

- Use `uri` when query or fragment state changes the screen identity.
- Use `fullPath` when analytics should group dynamic IDs.
- Use `pathParameters` for typed values instead of splitting URLs.
- Test `/catalog`, `/catalog/item/42`, and `/catalog?filter=sale`.
- Test a Web deep link on a cold start, not only an in-app tab tap.

`matchedLocation` gives route-aware widgets a narrow contract: concrete path for navigation structure, complete URI for address state, and full path for analytics templates. Keeping those responsibilities separate prevents nested routes and filters from making the Flutter shell look lost.

Reference: [GoRouterState API](https://pub.dev/documentation/go_router/latest/go_router/GoRouterState-class.html) and [go_router configuration](https://pub.dev/documentation/go_router/latest/topics/Configuration-topic.html).
