---
layout: post
title: "go_router optionURLReflectsImperativeAPIs - Keep Flutter Web URLs in Sync with push()"
description: "Learn how go_router optionURLReflectsImperativeAPIs changes Flutter Web URL updates for push(), pop(), browser back, and nested navigation."
date: 2026-08-26
tags: [go_router, navigation, FlutterWeb, Web, testing]
comments: true
share: true
---

![go_router keeping imperative Flutter navigation in sync with the browser URL](/assets/images/go-router-stateful-shell-route.png)

`go_router` normally treats `go()` and `push()` differently on Flutter Web. `go()` replaces the current route location and updates the address bar, while `push()` adds a page to the Navigator stack without necessarily making that imperative navigation visible in the browser URL. Setting `GoRouter.optionURLReflectsImperativeAPIs` to `true` changes that policy globally.

That setting is useful when a detail page opened with `context.push()` should survive a browser refresh, be copied as a link, or appear in browser history. It is not a universal upgrade, though. A temporary dialog-like page or an internal flow can become part of the public URL when you enable it.

## The default difference between go() and push()

Consider a small catalog route:

```dart
final router = GoRouter(
  routes: [
    GoRoute(
      path: '/',
      builder: (context, state) => const CatalogPage(),
      routes: [
        GoRoute(
          path: 'products/:id',
          builder: (context, state) => ProductPage(
            id: state.pathParameters['id']!,
          ),
        ),
      ],
    ),
  ],
);
```

The two common calls express different navigation intent:

| Call | Navigator behavior | Typical URL meaning |
| --- | --- | --- |
| `context.go('/products/42')` | Replaces the current location | The URL describes the current app state |
| `context.push('/products/42')` | Adds a page above the current stack | Often a temporary, imperative destination |
| `context.pop()` | Removes the top page | Returns to the previous stack entry |

The distinction is easy to miss because both `go()` and `push()` show the same widget. The difference appears when the user presses browser refresh, copies the URL, or uses the browser Back button. On Web, a navigation policy is also a URL policy.

## Enable URL reflection for imperative APIs

Set the static option before creating the router. Keeping it beside router initialization makes the global behavior obvious:

```dart
void main() {
  GoRouter.optionURLReflectsImperativeAPIs = true;

  runApp(
    MaterialApp.router(
      routerConfig: appRouter,
    ),
  );
}

final appRouter = GoRouter(
  routes: [
    GoRoute(
      path: '/',
      builder: (context, state) => const CatalogPage(),
      routes: [
        GoRoute(
          path: 'products/:id',
          builder: (context, state) => ProductPage(
            id: state.pathParameters['id']!,
          ),
        ),
      ],
    ),
  ],
);
```

Now this call is reflected by the browser location:

```dart
onTap: () => context.push('/products/42'),
```

The visible result is still a pushed page, so `pop()` returns to the previous page in the Navigator stack. The important extra behavior is that the address bar also represents the imperative route. A copied `/products/42` can be opened as a deep link, provided the route does not depend on in-memory data that exists only in the original page.

The option is static, so it is not a per-route switch. Do not try to configure it inside a `GoRoute` or expect two routers in the same application to use different policies. If one section needs public URLs and another section needs private stack entries, choose the navigation API deliberately or split the behavior at a higher architectural boundary.

## URL reflection is not the same as serializing route data

A URL can contain the path and query parameters, but it cannot automatically preserve every object passed through `extra`:

```dart
context.push(
  '/products/42',
  extra: Product(
    id: '42',
    title: 'Keyboard',
  ),
);
```

With URL reflection enabled, `/products/42` can be written to the address bar. The `Product` object is still an in-memory value. A refresh may reconstruct the route from the URL without reconstructing that exact object. The route should therefore treat the path parameter as the durable source of identity and reload the product from a repository.

```dart
class ProductPage extends StatelessWidget {
  const ProductPage({required this.id, super.key});

  final String id;

  @override
  Widget build(BuildContext context) {
    return ProductLoader(productId: id);
  }
}
```

If a short-lived object really must travel through browser history, configure an appropriate `extraCodec` and test browser restoration. Even then, I keep shareable state in path or query parameters. The URL should remain understandable without knowing the internal object graph of the app.

## What changes with browser Back?

When imperative navigation is reflected, browser history and Navigator history become connected. A sequence such as this deserves an explicit test:

```dart
context.push('/products/42');
context.push('/products/42/reviews');
```

The user may then press browser Back, use an in-app back button, or refresh at the reviews URL. These actions should converge on the same route contract. Problems usually appear when a page assumes that it was only reached from a particular parent page.

| Scenario | Safe expectation |
| --- | --- |
| Refresh `/products/42` | Load product `42` from durable data |
| Open `/products/42` directly | Do not require a previous catalog page |
| Browser Back from reviews | Return to the prior browser location |
| `context.pop()` from reviews | Remove the top Navigator page |
| Missing product ID | Show a route-level error or not-found state |

For a bottom-navigation layout, test this with the real `ShellRoute` or `StatefulShellRoute`. Each branch can have its own Navigator, and the resulting URL history may not match a simplified single-Navigator test. A bug that looks like a browser problem is often a mismatch between shell navigation and child navigation.

## When should you enable it?

I enable `optionURLReflectsImperativeAPIs` when pushed pages represent meaningful application state: product details, search results, article pages, or a multi-step screen that users may bookmark. I leave it disabled when `push()` is being used like a transient overlay, an onboarding checkpoint, or a private internal step that should disappear without becoming a shareable location.

The decision can be made with three questions:

- Should a refresh reopen this destination?
- Should a copied URL be useful to another user?
- Can the destination rebuild itself from path and query data?

If the answer is “yes” to all three, URL reflection is usually a good fit. If the page only makes sense because an earlier widget owns unsaved memory, keep that state out of the address bar and use ordinary stack navigation.

`GoRouter.optionURLReflectsImperativeAPIs` is a small global switch with a large architectural effect. It changes `push()` from a navigation action that is mostly local to a browser-visible state change. Enable it alongside durable route parameters, direct-link tests, and a clear policy for shell routes; otherwise the address bar may promise more restoration than the app can actually provide.

References: [GoRouter API](https://pub.dev/documentation/go_router/latest/go_router/GoRouter-class.html), [go_router navigation topic](https://pub.dev/documentation/go_router/latest/topics/Navigation-topic.html).
