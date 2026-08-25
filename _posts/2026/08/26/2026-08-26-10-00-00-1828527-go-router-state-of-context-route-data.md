---
layout: post
title: "go_router GoRouterState.of - Read Flutter Route Data Without Passing Props"
description: "Learn how GoRouterState.of reads Flutter route data inside nested widgets, with safe URI parsing, refresh behavior, and testing patterns."
date: 2026-08-26
tags: [go_router, navigation, Flutter, testing, performance]
comments: true
share: true
---

![go_router reading route state inside a nested Flutter page](/assets/images/go-router-extra-typed-route-data.png)

`GoRouterState.of(context)` is useful when a Flutter widget needs the current route URI, path parameters, or query parameters but does not own the `GoRoute` builder. It removes a layer of prop drilling, while keeping the URL as the source of truth. The important boundary is that the widget must be below the route in the widget tree; calling it from an unrelated context is not a substitute for passing application data.

## The problem with passing route data through every widget

I initially passed an order ID from the `GoRoute` builder into the page, then from the page into the toolbar, then into a share button. That worked until the route gained a query parameter for the selected tab. One widget was rebuilt with the new URL, while another still held the old constructor value.

There are two different kinds of data here:

| Data | Good source | Why |
| --- | --- | --- |
| `/orders/42` | `state.pathParameters` | Identifies the resource in the URL |
| `?tab=history` | `state.uri.queryParameters` | Represents shareable view state |
| Loaded order model | Repository or state management | Not every model belongs in the URL |

Read the first two from route state. Load the third from an application data source using the ID.

## Read the current URI from a descendant widget

The following route keeps its page API small. `OrderActions` reads the current route only where it needs it:

```dart
final router = GoRouter(
  routes: [
    GoRoute(
      path: '/orders/:orderId',
      builder: (context, state) {
        return const OrderPage();
      },
    ),
  ],
);

class OrderPage extends StatelessWidget {
  const OrderPage({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Order')),
      body: const OrderActions(),
    );
  }
}

class OrderActions extends StatelessWidget {
  const OrderActions({super.key});

  @override
  Widget build(BuildContext context) {
    final routeState = GoRouterState.of(context);
    final orderId = routeState.pathParameters['orderId'];

    if (orderId == null || orderId.isEmpty) {
      return const Text('The order link is invalid.');
    }

    return FilledButton(
      onPressed: () => ShareService.shareOrder(orderId),
      child: Text('Share order $orderId'),
    );
  }
}
```

The route builder still decides which page to show, but the nested action does not need a constructor parameter that exists only to repeat route information. This is especially handy for a persistent app bar, a nested tab, or a reusable action several levels below the route.

## Prefer `uri` when the widget needs query state

`GoRouterState.of(context)` exposes the complete `Uri`. I prefer reading query parameters from `uri` rather than treating the raw location as a string:

```dart
class OrderFilterLabel extends StatelessWidget {
  const OrderFilterLabel({super.key});

  @override
  Widget build(BuildContext context) {
    final uri = GoRouterState.of(context).uri;
    final tab = uri.queryParameters['tab'] ?? 'summary';
    final label = switch (tab) {
      'history' => 'History',
      'invoices' => 'Invoices',
      _ => 'Summary',
    };

    return Text(label);
  }
}
```

This handles encoded values and keeps path parsing inside `go_router`. It also exposes a useful limit: reading the query parameter does not automatically update a separate controller. The widget rebuilds when the route subtree is rebuilt, but a long-lived service should not quietly depend on `BuildContext`.

## Do not use route state as a model store

A path parameter is normally a key, not the complete object. I use it to start a repository lookup:

```dart
class OrderBody extends StatelessWidget {
  const OrderBody({super.key});

  @override
  Widget build(BuildContext context) {
    final orderId = GoRouterState.of(context).pathParameters['orderId'];

    if (orderId == null) {
      return const Center(child: Text('Missing order ID'));
    }

    return OrderView(orderId: orderId);
  }
}
```

Putting a full order object in `extra` can make an in-app transition look convenient, but refreshes, browser restores, and copied URLs may not carry that object. A stable ID in the path survives those boundaries. Keep credentials, tokens, and large payloads out of both the URL and `extra`.

## The context boundary is the common trap

This call fails when the context is above `MaterialApp.router` or outside the matched route subtree. A service, a top-level provider, or a global singleton should not call `GoRouterState.of` just because it needs navigation data. Pass a small value into that layer, or let the layer observe an explicit application state object.

The same issue appears with dialogs. A dialog opened from a route usually receives a context below the route, but a dialog launched from a root overlay may not. Capture the needed ID before opening it when the ownership is unclear:

```dart
void openCancelDialog(BuildContext context) {
  final orderId = GoRouterState.of(context).pathParameters['orderId'];
  if (orderId == null) return;

  showDialog<void>(
    context: context,
    builder: (_) => AlertDialog(
      title: const Text('Cancel order?'),
      actions: [
        TextButton(
          onPressed: () => Navigator.of(context).pop(),
          child: Text('Cancel $orderId'),
        ),
      ],
    ),
  );
}
```

## Test the route, not only the widget

A widget test that pumps `OrderActions` by itself has no `GoRouter` ancestor, so `GoRouterState.of(context)` cannot resolve. Build the actual router and navigate to the path under test:

```dart
testWidgets('reads the order ID from the route', (tester) async {
  final router = GoRouter(
    initialLocation: '/orders/42',
    routes: [
      GoRoute(
        path: '/orders/:orderId',
        builder: (_, __) => const OrderActions(),
      ),
    ],
  );

  await tester.pumpWidget(MaterialApp.router(routerConfig: router));
  await tester.pumpAndSettle();

  expect(find.text('Share order 42'), findsOneWidget);
});
```

Also test a missing parameter, an encoded ID, and a query change. Those cases reveal whether the widget is validating route input or accidentally assuming that every URL was generated by your own button.

`GoRouterState.of(context)` is a focused dependency: use it at the UI boundary that needs route identity, parse defensively, and keep domain models in repositories or state management. That separation makes deep links, browser refreshes, and nested widgets behave consistently without turning every constructor into a copy of the route definition.
