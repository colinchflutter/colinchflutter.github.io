---
layout: post
title: "go_router URI Migration - Fix Flutter Deep-Link Regressions After state.queryParameters"
description: "Migrate Flutter go_router state fields to Uri safely, preserve path and query values, and verify deep links after a production upgrade."
date: 2026-09-16
tags: [go_router, migration, navigation, Flutter, FlutterWeb, testing]
comments: true
share: true
---

![go_router URI migration and Flutter deep-link verification](/assets/images/go-router-stateful-shell-route.png)

If a Flutter app still reads `state.queryParameters`, the immediate fix is `state.uri.queryParameters`, but a search-and-replace alone is not a safe migration. Use this guide when upgrading an existing `go_router` app that has web URLs, filters, or path-based detail screens. A mobile-only app that never restores or shares URLs can stop after the compile fix; a web or deep-linkable app should also verify encoding, path semantics, and browser refresh.

## What changed and what did not

`go_router` 10.0.0 replaced the separate location and query fields on `GoRouterState` with one `Uri`. The current package still documents path and query parameters as separate URL concepts, so the migration is not “put everything in query parameters.” Keep route identity in the path and optional screen state in the query.

| Old code | Migrated code | Meaning |
| --- | --- | --- |
| `state.location` | `state.uri.toString()` | Full location, including query and fragment |
| `state.queryParameters['q']` | `state.uri.queryParameters['q']` | One decoded query value |
| `state.queryParametersAll['tag']` | `state.uri.queryParametersAll['tag']` | Repeated query values |
| `state.pathParameters['id']` | `state.pathParameters['id']` | Path parameters remain path parameters |

The last row is the trap. Do not move `:id` into `?id=` just to make the new API look uniform.

## A migration boundary that is easy to review

Keep URL reading in a small adapter. It gives code review one place to check null handling and makes route tests independent from page widgets.

```dart
class ProductLocation {
  const ProductLocation({required this.id, this.search = ''});

  final String id;
  final String search;

  factory ProductLocation.fromState(GoRouterState state) {
    final id = state.pathParameters['id'];
    if (id == null || id.isEmpty) {
      throw const FormatException('Missing product id');
    }

    return ProductLocation(
      id: id,
      search: state.uri.queryParameters['q'] ?? '',
    );
  }
}

GoRoute(
  path: '/products/:id',
  builder: (context, state) {
    final location = ProductLocation.fromState(state);
    return ProductPage(productId: location.id, search: location.search);
  },
),
```

For a redirect or analytics callback, use `state.uri.path` when comparing only the path. Comparing `state.uri.toString()` to `'/products/42'` will fail as soon as a query or fragment is present.

```dart
redirect: (context, state) {
  final isProduct = state.uri.path.startsWith('/products/');
  final hasSession = session.isSignedIn;

  if (isProduct && !hasSession) {
    final from = Uri.encodeComponent(state.uri.toString());
    return '/login?from=$from';
  }
  return null;
},
```

## Regression checks worth keeping

The migration is complete only when the same URL behaves correctly on a cold start, an in-app navigation, and a browser refresh. These cases catch different failures.

| Case | Expected assertion |
| --- | --- |
| `/products/42?q=red%20shoes` | `id == '42'`, `search == 'red shoes'` |
| `/products/42?q=a&q=b` | `queryParametersAll['q'] == ['a', 'b']` |
| `/products/42#reviews` | `state.uri.path == '/products/42'`, fragment is separate |
| missing `/products/:id` | error route or controlled redirect, never `!` crash |

Copy this checklist into the upgrade PR:

- [ ] Replace removed state fields and search for `state.location`, `queryParametersAll`, and old `queryParams` names.
- [ ] Use `uri.path` for path-only comparisons and `uri.toString()` only when the full URL is intended.
- [ ] Test percent-encoded values and repeated query keys.
- [ ] Open one deep link directly in a release web build and refresh it.
- [ ] Confirm login redirects preserve the internal path and query, not an arbitrary external URL.

## Short takeaway

`GoRouterState` now has a single URI source, but URL design still has separate responsibilities. Migrate reads to `state.uri`, leave resource identity in `pathParameters`, and make deep-link tests part of the package upgrade. The official changelog records the breaking change and the later fix for query parameters during refresh; check the package version and migration guide used by your project before applying an older snippet.

References: [go_router changelog](https://pub.dev/packages/go_router/changelog), [go_router URI migration guide](https://docs.flutter.dev/go/go-router-v10-breaking-changes), and [go_router upgrading topic](https://pub.dev/documentation/go_router/latest/topics/Upgrading-topic.html).
