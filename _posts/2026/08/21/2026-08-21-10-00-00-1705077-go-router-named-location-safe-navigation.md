---
layout: post
title: "go_router namedLocation - Build Safer Flutter URLs Without String Concatenation"
description: "Learn how to use go_router namedLocation and goNamed in Flutter to generate safe URLs, preserve query parameters, and avoid broken navigation strings."
date: 2026-08-21
tags: [go_router, navigation, testing, Web]
comments: true
share: true
---

![go_router namedLocation generating a safe Flutter navigation URL](/assets/images/go-router-extra-typed-route-data.png)

`go_router` named routes are useful when a Flutter app has more than a handful of screens, but the real benefit is not the route name itself. `namedLocation` gives the router ownership of URL construction, so path segments and query parameters are encoded in one place instead of being assembled with fragile string concatenation.

## Why route strings become a maintenance problem

This looks harmless:

```dart
context.go('/products/${product.id}?tab=reviews');
```

It becomes risky when a route changes from `/products/:id` to `/catalog/:id`, when `tab` needs URL encoding, or when a required parameter is accidentally omitted. The compiler cannot help with any of those mistakes because the whole destination is just a string.

Named routes make the route contract visible in the router:

| Route concern | String concatenation | Named navigation |
|---|---|---|
| Path rename | Search every caller | Change the route definition and callers can follow it |
| Required ID | Easy to omit | Required by `pathParameters` |
| Search/filter state | Manual `?key=value` | Built by `queryParameters` |
| Deep-link testing | Compare hand-written strings | Test one URL builder |

## Define a route with a stable name

The route name is an internal identifier. Keep it stable even if the visible URL changes later.

```dart
final router = GoRouter(
  initialLocation: '/products',
  routes: [
    GoRoute(
      path: '/products',
      name: 'products',
      builder: (context, state) => ProductListPage(
        search: state.uri.queryParameters['q'] ?? '',
      ),
      routes: [
        GoRoute(
          path: ':productId',
          name: 'product',
          builder: (context, state) => ProductPage(
            id: state.pathParameters['productId']!,
          ),
        ),
      ],
    ),
  ],
);
```

The child route name is still globally addressable. Its path is nested under `/products`, but callers do not need to know how that nesting is represented.

## Generate a URL with namedLocation

Use `namedLocation` when a URL is needed before navigation, such as for a share button, a browser link, or a test assertion.

```dart
final productUrl = router.namedLocation(
  'product',
  pathParameters: {'productId': '42'},
  queryParameters: {'ref': 'email'},
);

debugPrint(productUrl);
// /products/42?ref=email
```

The same route can be opened directly with `goNamed`:

```dart
context.goNamed(
  'product',
  pathParameters: {'productId': product.id.toString()},
  queryParameters: {'ref': 'catalog'},
);
```

`goNamed` is a navigation action. `namedLocation` only produces the location string. Keeping those uses separate helps when a widget needs to copy or share a URL without changing the current screen.

## Keep path and query data separate

A path parameter identifies the resource. A query parameter describes the current view of that resource. Mixing the two makes URLs harder to reason about.

```dart
final url = router.namedLocation(
  'products',
  queryParameters: {
    if (search.trim().isNotEmpty) 'q': search.trim(),
    if (selectedCategory != null) 'category': selectedCategory!,
  },
);

context.go(url);
```

`Uri` encoding is handled by the router, so a search such as `red shoes` becomes a valid URL rather than a location containing an unsafe literal space. Empty filters should usually be omitted instead of serialized as `q=`. That keeps copied links shorter and makes default state unambiguous.

## A small helper prevents duplicated route contracts

The `GoRouter` instance can be hidden behind a route helper when several features link to the same page.

```dart
abstract final class ProductRoute {
  static String detailsUrl(GoRouter router, String id) {
    if (id.isEmpty) {
      throw ArgumentError.value(id, 'id', 'Product ID cannot be empty');
    }

    return router.namedLocation(
      'product',
      pathParameters: {'productId': id},
    );
  }
}
```

The validation is intentionally close to URL generation. A missing ID should fail while building the link, not after a user taps it and reaches a generic error page.

## Common mistakes

Passing a parent route name when generating a child URL is a frequent error. The `product` name above is the destination; `products` would generate the collection route and cannot accept `productId`.

Another trap is using `goNamed` for every interaction. `goNamed` replaces the current location, which is usually right for tabs and canonical pages. A modal detail flow may need `pushNamed` so the back button returns to the previous page:

```dart
context.pushNamed(
  'product',
  pathParameters: {'productId': product.id.toString()},
);
```

Finally, route names are not compile-time checked by `go_router` itself. A typo such as `'prodcut'` still compiles. Put route-name helpers in one file and add a test for every public URL builder.

```dart
test('product URL contains the encoded ID', () {
  final location = router.namedLocation(
    'product',
    pathParameters: {'productId': 'red/42'},
  );

  expect(location, '/products/red%2F42');
});
```

The practical rule is simple: use `goNamed` or `pushNamed` for navigation, use `namedLocation` for links and assertions, and let the router assemble every path and query segment. This keeps Flutter deep links readable while reducing the number of invisible string contracts spread across the app.
