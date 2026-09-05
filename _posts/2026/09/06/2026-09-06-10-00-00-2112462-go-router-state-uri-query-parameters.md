---
layout: post
title: "go_router GoRouterState.uri - Read Flutter Query Parameters Safely"
description: "Learn how GoRouterState.uri reads Flutter query parameters, preserves repeated values, and avoids fragile URL parsing in route builders."
date: 2026-09-06
tags: [go_router, navigation, Web, testing]
comments: true
share: true
---

![go_router reading Flutter route state and query parameters](/assets/images/go-router-extra-typed-route-data.png)

`go_router` query parameters should be read from `GoRouterState.uri`, not by splitting `state.location` or manually parsing a browser URL. `state.uri.queryParameters` is enough for ordinary filters, while `queryParametersAll` is the safer choice when a URL can contain repeated keys such as `tag=flutter&tag=router`.

I initially treated query parameters like a small string-formatting problem. That worked for `/search?q=flutter`, but it broke as soon as a filter was repeated or a value was missing. The route builder already receives a parsed `Uri`; using it keeps URL decoding and route matching in one place.

## The route state to inspect

Here is a small search route. The query parameter is read inside `builder`, so the widget is rebuilt with the current route state after a new URL is applied.

```dart
import 'package:flutter/material.dart';
import 'package:go_router/go_router.dart';

final router = GoRouter(
  routes: [
    GoRoute(
      path: '/search',
      builder: (context, state) {
        final query = state.uri.queryParameters['q'] ?? '';
        final sort = state.uri.queryParameters['sort'] ?? 'relevance';

        return SearchPage(
          query: query,
          sort: sort,
        );
      },
    ),
  ],
);
```

For `/search?q=flutter%20router&sort=recent`, `Uri` decodes the values before they reach the widget. There is no reason to call `split('=')`, and doing so can lose data when a value itself contains `=` or an encoded character.

## Single values and repeated values are different contracts

The most common mistake is assuming every query key has exactly one value. `queryParameters` returns a `Map<String, String>`, so repeated values are collapsed to one value. When the URL represents a multi-select filter, that is the wrong data shape.

| URL shape | API | Result | Suitable for |
| --- | --- | --- | --- |
| `?q=flutter` | `queryParameters['q']` | `flutter` | Search text, sort order |
| `?tag=flutter&tag=router` | `queryParametersAll['tag']` | `['flutter', 'router']` | Multi-select filters |
| missing `?page=` | `queryParameters['page']` | `null` or empty fallback | Optional state |
| `?page=abc` | `int.tryParse(...)` | `null` | Untrusted numeric input |

For a filter page, I prefer a small parser that validates the input at the route boundary:

```dart
class SearchRouteData {
  const SearchRouteData({
    required this.query,
    required this.tags,
    required this.page,
  });

  final String query;
  final List<String> tags;
  final int page;

  factory SearchRouteData.fromUri(Uri uri) {
    final requestedPage = int.tryParse(uri.queryParameters['page'] ?? '');
    final page = requestedPage != null && requestedPage > 0
        ? requestedPage
        : 1;

    final tags = (uri.queryParametersAll['tag'] ?? const <String>[])
        .where((tag) => tag.trim().isNotEmpty)
        .toList(growable: false);

    return SearchRouteData(
      query: uri.queryParameters['q']?.trim() ?? '',
      tags: tags,
      page: page,
    );
  }
}
```

The route becomes easier to test when parsing is separated from the page widget:

```dart
GoRoute(
  path: '/search',
  builder: (context, state) {
    final data = SearchRouteData.fromUri(state.uri);
    return SearchPage(data: data);
  },
),
```

This also gives the application one clear policy for malformed URLs. A user can type `?page=-4` or follow an old bookmark with an invalid value; the page should not crash because the address bar is untrusted input.

## `uri` versus path parameters

`state.uri` contains the complete parsed URI, including the path and query string. A path parameter still belongs in `state.pathParameters` because it is part of the route pattern:

```dart
GoRoute(
  path: '/products/:productId',
  builder: (context, state) {
    final productId = state.pathParameters['productId']!;
    final tab = state.uri.queryParameters['tab'] ?? 'overview';

    return ProductPage(
      productId: productId,
      initialTab: tab,
    );
  },
),
```

I use this boundary consistently: the product ID identifies the resource, while `tab` controls the current view of that resource. Putting both values into an ad-hoc parser makes deep links harder to reason about and test.

## Avoid rebuilding a URL by hand

When changing filters, construct a `Uri` and pass its string to `go`. This preserves escaping for spaces, ampersands, and non-ASCII text.

```dart
void openSearch(BuildContext context, String text, List<String> tags) {
  final uri = Uri(
    path: '/search',
    queryParametersAll: {
      'q': [text],
      'page': ['1'],
      if (tags.isNotEmpty) 'tag': tags,
    },
  );

  context.go(uri.toString());
}
```

For a real filter model, I would build `queryParametersAll` once rather than repeatedly parsing the intermediate string. The important point is the contract: URL encoding belongs to `Uri`, and navigation receives the final serialized location.

## A focused test for route parsing

The parser can be tested without pumping a Flutter widget or starting a browser:

```dart
import 'package:flutter_test/flutter_test.dart';

void main() {
  test('preserves repeated tags and normalizes an invalid page', () {
    final data = SearchRouteData.fromUri(
      Uri.parse('/search?q=flutter%20router&tag=flutter&tag=web&page=0'),
    );

    expect(data.query, 'flutter router');
    expect(data.tags, ['flutter', 'web']);
    expect(data.page, 1);
  });
}
```

The practical rule is small: use `state.pathParameters` for route identity, `state.uri.queryParameters` for one-value options, and `state.uri.queryParametersAll` for repeated filters. Parse and validate at the route boundary, then pass typed data into the page. That keeps Flutter Web refreshes, copied deep links, and ordinary in-app navigation on the same code path.
