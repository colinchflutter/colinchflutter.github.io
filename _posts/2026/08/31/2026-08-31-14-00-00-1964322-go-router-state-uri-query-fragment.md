---
layout: post
title: "go_router GoRouterState.uri - Restore Flutter Filter State from Deep Links"
description: "Learn how GoRouterState.uri preserves Flutter path, query, and fragment state for shareable filters and reliable deep links."
date: 2026-08-31
tags: [go_router, navigation, Web, state_management]
comments: true
share: true
---

![go_router GoRouterState.uri carrying Flutter route and query state](/assets/images/go-router-extra-typed-route-data.png)

`GoRouterState.uri` is the safest single source for the complete URL of a Flutter route. I use it when a screen must restore a search query, a selected filter, or a browser fragment without combining several partially decoded route fields. The important detail is that `uri` is a `Uri`, not a raw string, so Dart handles query decoding and repeated values for us.

## Why `state.location`-style code becomes fragile

Older examples often read a location string and split it manually. That breaks quickly when a query contains encoded spaces, multiple values, or a fragment:

| Route data | Recommended API | Typical use |
| --- | --- | --- |
| `/products/42` | `state.pathParameters` | Resource identity |
| `?brand=acme&brand=globex` | `state.uri.queryParametersAll` | Repeated filters |
| `#reviews` | `state.uri.fragment` | In-page anchor |
| Complete URL | `state.uri` | Canonical screen state |

The current API exposes `uri` as the full URI, for example `/family/f2/person/p1?filter=name#details`. That makes the route state explicit and keeps parsing at the routing boundary.

## Parse a shareable product filter

The route builder can convert URL values into a small immutable view model. This code keeps malformed values harmless and supports a filter that appears more than once.

```dart
class ProductFilter {
  const ProductFilter({this.brands = const [], this.sort = 'popular'});

  final List<String> brands;
  final String sort;

  factory ProductFilter.fromUri(Uri uri) {
    final allowedSorts = {'popular', 'price-low', 'newest'};
    final sort = uri.queryParameters['sort'];

    return ProductFilter(
      brands: uri.queryParametersAll['brand']
          ?.where((value) => value.isNotEmpty)
          .toList(growable: false) ??
          const [],
      sort: allowedSorts.contains(sort) ? sort! : 'popular',
    );
  }
}

GoRoute(
  path: '/products',
  builder: (context, state) {
    final filter = ProductFilter.fromUri(state.uri);
    return ProductPage(filter: filter, anchor: state.uri.fragment);
  },
),
```

The parser is intentionally separate from the widget. A browser refresh, a notification deep link, and an in-app `context.go` now produce the same `ProductFilter`. The page does not need to know whether the value came from Android, iOS, or Flutter Web.

## Update the URL without losing existing state

When a user changes one filter, construct a new `Uri` from the current state instead of concatenating strings. In a real page, keep the current `Uri` in the route-aware widget or derive it from `GoRouter.of(context).routeInformationProvider.value.uri` before navigating.

```dart
void setSort(BuildContext context, String sort) {
  final current = GoRouterState.of(context).uri;
  final query = <String, dynamic>{
    for (final entry in current.queryParametersAll.entries)
      entry.key: entry.value.length == 1 ? entry.value.first : entry.value,
  }..['sort'] = sort;

  context.go(current.replace(queryParameters: query).toString());
}
```

`queryParameters` is convenient for single-value keys. For repeated keys such as `brand`, pass an `Iterable<String>` as the map value when constructing the replacement URI so the second selected brand does not overwrite the first one. Keep the fragment when the user is changing a filter unless the interaction intentionally moves the anchor.

## Traps I check before shipping

- `queryParameters['brand']` returns only one value. Use `queryParametersAll` for checkbox-style filters.
- An absent key and an empty key are different inputs. Decide whether `?sort=` means the default or an invalid value.
- Do not pass the entire `Uri` into a repository as business data. Parse and validate it at the route boundary.
- URL state should stay small and non-sensitive. Tokens, private search terms, and large JSON objects do not belong in a shareable location.
- Test a browser refresh and a copied URL, not only a button tap. Those paths reveal missing defaults and encoding mistakes.

The useful rule is simple: use `pathParameters` for identity, `queryParametersAll` for repeatable screen options, and `fragment` for a visual anchor. `GoRouterState.uri` keeps those pieces together, which makes Flutter deep links easier to reproduce, test, and share.
