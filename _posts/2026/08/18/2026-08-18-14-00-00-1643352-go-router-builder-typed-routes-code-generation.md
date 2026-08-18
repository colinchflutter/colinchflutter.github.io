---
layout: post
title: "go_router_builder Typed Routes - Generate Safer Flutter Navigation Code"
description: "Use go_router_builder typed routes to generate Flutter navigation helpers, catch parameter mistakes at compile time, and keep deep links explicit."
date: 2026-08-18
tags: [go_router, navigation, testing, Web]
comments: true
share: true
---

![go_router_builder turning a typed Flutter route declaration into a detail screen and deep link](/assets/images/go-router-extra-typed-route-data.png)

This picture shows the useful boundary in `go_router_builder`: a route declaration is transformed into navigation code and a URL, while the screen still receives typed values. The generated code does not remove routing decisions, but it does remove a large class of stringly typed mistakes.

## The problem with string routes

Plain `go_router` is easy to start with. A product detail page might be opened like this:

```dart
context.go('/products/$productId');
```

That line hides several assumptions. The path must match the route declaration, `productId` must be encoded correctly, and every caller has to know the route's spelling. A rename from `/products/:id` to `/items/:id` can compile successfully while leaving broken navigation in another feature.

Typed routes move those assumptions into Dart classes. The route's required parameters become constructor parameters, so the compiler can point at an incomplete or incorrectly typed navigation call.

## Define the route once

Add the generator packages and create a route file. The `part` directive is required because the builder writes the router list and helper extensions into a generated Dart file.

```yaml
dependencies:
  go_router: any

dev_dependencies:
  build_runner: any
  go_router_builder: any
```

The route class below keeps the URL parameter as a `String`, which is usually the safest choice for IDs coming from a browser address bar.

```dart
import 'package:flutter/material.dart';
import 'package:go_router/go_router.dart';
import 'package:go_router_builder/go_router_builder.dart';

part 'app_routes.g.dart';

@TypedGoRoute<HomeRoute>(path: '/')
class HomeRoute extends GoRouteData {
  const HomeRoute();

  @override
  Widget build(BuildContext context, GoRouterState state) {
    return const ProductListScreen();
  }
}

@TypedGoRoute<ProductRoute>(path: '/products/:id')
class ProductRoute extends GoRouteData {
  const ProductRoute({required this.id});

  final String id;

  @override
  Widget build(BuildContext context, GoRouterState state) {
    return ProductDetailScreen(productId: id);
  }
}

final router = GoRouter(routes: $appRoutes);
```

Generate the missing file with the project-level command:

```bash
dart run build_runner build --delete-conflicting-outputs
```

The generated `$appRoutes` value becomes the source of truth for `GoRouter`. The important detail is that generated files are build output. They should not be edited by hand; change the route declaration and run the builder again.

## Navigate with the generated API

Once the generator has run, navigation can use the route object instead of manually assembling a path.

```dart
class ProductTile extends StatelessWidget {
  const ProductTile({required this.productId, super.key});

  final String productId;

  @override
  Widget build(BuildContext context) {
    return ListTile(
      title: Text(productId),
      onTap: () => ProductRoute(id: productId).go(context),
    );
  }
}
```

For a stack-style transition, use `push` instead of `go`:

```dart
ProductRoute(id: productId).push(context);
```

The route object also exposes a generated location, which is useful for a link or a test that should inspect the final URL:

```dart
final location = ProductRoute(id: 'sku-42').location;
// /products/sku-42
```

This is the practical benefit: a missing `id` is a Dart error, and the URL format is generated from the same declaration used by the router.

## Typed does not mean every value belongs in the path

Path parameters are a good fit for identity. They are less suitable for temporary UI state or large objects. Keep the boundary explicit:

| Value | Recommended route data | Reason |
|---|---|---|
| Product ID | Path parameter | Survives refresh and copied links |
| Search text | Query parameter | Describes shareable page state |
| Loaded product object | `extra` or repository lookup | Avoids putting a large object in the URL |
| Login-only decision | Redirect | Must be reevaluated when auth changes |

For example, a detail page can receive an ID from the typed route and load the current product from a repository. That is slightly more work than passing the whole object, but it behaves correctly when a user opens `/products/sku-42` directly in a new browser tab.

## Traps I would check before adopting it

The generator adds a build step to every route change. In a team project, CI should run `dart run build_runner build --delete-conflicting-outputs` or check in generated files according to the repository's convention. A route annotation without regenerated output can look correct in an editor and still fail in a clean build.

Nested routes also need deliberate structure. A typed child route should be declared under its typed parent when it depends on the parent's shell or parameters. If every screen is declared as a top-level route, the code may be typed while the visual navigation hierarchy is still wrong.

I would keep typed routes for stable application destinations and use ordinary `GoRoute` for highly dynamic, configuration-driven route trees. Code generation is valuable when the route graph is maintained by Dart code; it is friction when the graph is data-driven.

## Short checklist

- Put stable identity in path parameters.
- Use generated `.go`, `.push`, and `.location` helpers instead of route strings.
- Regenerate after changing annotations.
- Test a direct deep link, not only taps from the previous screen.
- Keep large objects and short-lived state outside the URL.

References: [go_router typed routes](https://pub.dev/documentation/go_router/latest/topics/Typed%20routing-topic.html), [go_router_builder on pub.dev](https://pub.dev/packages/go_router_builder).
