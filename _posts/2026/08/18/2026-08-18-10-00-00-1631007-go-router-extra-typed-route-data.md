---
layout: post
title: "go_router extra - Pass Typed Route Data in Flutter Without Breaking Deep Links"
description: "Learn when to use go_router extra for typed Flutter route data, and when path parameters are safer for deep links, refreshes, and web navigation."
date: 2026-08-18
tags: [go_router, navigation, state_management, Web, Flutter]
comments: true
share: true
---

![go_router passing typed route data from a Flutter product list to a detail screen](/assets/images/go-router-extra-typed-route-data.png)

`go_router`'s `extra` is a convenient way to pass a typed Dart object from one Flutter route to another. It is a good fit for an in-memory handoff, such as opening a product detail page from a list that already loaded the product. It is not a replacement for path parameters when the destination must survive a browser refresh, a cold start, or a copied deep link.

I initially used `extra` for every detail route because the code was short. The problem appeared on Flutter Web: clicking a product worked, but refreshing `/products/42` had no product object to restore. The route existed, while the data that the page needed did not. The fix is to make the URL the durable identity and treat `extra` as an optional optimization.

## Use `extra` for an in-memory object handoff

Suppose the list already has a parsed `Product`. Passing that object avoids decoding the same data immediately in the detail page.

```dart
class Product {
  const Product({
    required this.id,
    required this.name,
    required this.price,
  });

  final String id;
  final String name;
  final double price;
}

final router = GoRouter(
  routes: [
    GoRoute(
      path: '/products',
      builder: (context, state) => const ProductListPage(),
      routes: [
        GoRoute(
          path: ':id',
          builder: (context, state) {
            final product = state.extra as Product?;
            return ProductDetailPage(
              productId: state.pathParameters['id']!,
              initialProduct: product,
            );
          },
        ),
      ],
    ),
  ],
);
```

The navigation call stays readable at the point where the object is available:

```dart
context.push('/products/${product.id}', extra: product);
```

The route still reads the `id` from the URL. That detail matters. The object can make the first frame fast, but the ID is what lets the page recover when `extra` is missing.

## Make the detail page work with or without `extra`

The safest pattern is to render the passed object immediately and fetch the canonical record when it is absent or stale. A small repository method keeps this fallback outside the router.

```dart
class ProductDetailPage extends StatefulWidget {
  const ProductDetailPage({
    required this.productId,
    this.initialProduct,
    super.key,
  });

  final String productId;
  final Product? initialProduct;

  @override
  State<ProductDetailPage> createState() => _ProductDetailPageState();
}

class _ProductDetailPageState extends State<ProductDetailPage> {
  late Product? product = widget.initialProduct;
  late Future<Product>? request =
      product == null ? repository.load(widget.productId) : null;

  @override
  Widget build(BuildContext context) {
    if (product != null) {
      return ProductView(product: product!);
    }

    return FutureBuilder<Product>(
      future: request,
      builder: (context, snapshot) {
        if (snapshot.hasError) {
          return ProductErrorView(productId: widget.productId);
        }
        if (!snapshot.hasData) {
          return const ProductLoadingView();
        }
        return ProductView(product: snapshot.data!);
      },
    );
  }
}
```

This gives a fast transition from a list and a valid cold-start behavior. In a real app, I would also compare `initialProduct.id` with `widget.productId` before trusting the object. A stale object can be worse than a short loading state when the list was updated while the user was browsing.

## `extra` versus path and query parameters

The decision becomes clearer when the route has to work outside the current navigation session.

| Data | Best location | Why |
| --- | --- | --- |
| Product ID | Path parameter | Identifies the resource in a shareable URL |
| Search term or sort order | Query parameter | Describes a restorable view state |
| Already loaded `Product` object | `extra` | Avoids an immediate duplicate lookup |
| Unsaved form draft | State manager or local storage | It is not a stable route identity |
| Authentication token | Secure session storage | Never expose secrets in the URL or `extra` |

The common mistake is placing a complete object in `extra` and assuming that `go_router` serializes it into the address bar. It does not. On a browser refresh, a direct URL launch, or some restoration flows, the destination may receive `null`. `extra` can also be awkward when a route is restored by a platform that only knows the location string.

## Validate `extra` instead of trusting a cast

`state.extra as Product?` is concise, but it can throw if another caller passes a different type. A small type check makes a shared route safer:

```dart
final extra = state.extra;
final initialProduct = extra is Product ? extra : null;
final id = state.pathParameters['id'];

if (id == null || id.isEmpty) {
  return const ProductErrorView(productId: 'missing');
}

return ProductDetailPage(
  productId: id,
  initialProduct: initialProduct,
);
```

Do not put credentials, payment details, or large response graphs into route extras. The object is still part of process memory, and it does not become a secure transport boundary just because the router carries it.

## Test the route in both directions

The route is not finished when tapping the list item works. I check the same destination through four entry points:

1. Push from the product list with a valid `Product` in `extra`.
2. Open `/products/42` directly with no `extra`.
3. Refresh the browser while the detail page is visible.
4. Pass a wrong type or an unknown ID and verify the recovery UI.

The useful rule is simple: keep the resource identity in `pathParameters`, keep small view state in query parameters, and use `extra` only to improve an already-valid navigation path. That separation makes `go_router` routes feel fast during normal taps without making deep links depend on a Dart object that may no longer exist.

References: [go_router navigation](https://pub.dev/documentation/go_router/latest/topics/Navigation-topic.html), [go_router route data](https://pub.dev/documentation/go_router/latest/go_router/GoRouterState-class.html).
