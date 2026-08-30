---
layout: post
title: "go_router GoRouteData.buildPage - Add Type-Safe Custom Transitions in Flutter"
description: "Learn how go_router GoRouteData.buildPage creates type-safe Flutter pages with stable keys, custom transitions, and predictable route identity."
date: 2026-08-30
tags: [go_router, navigation, animation, testing]
comments: true
share: true
---

![go_router GoRouteData building a type-safe custom Flutter page](/assets/images/go-router-extra-typed-route-data.png)

`GoRouteData.buildPage` is the cleanest place to customize a generated `go_router` page in Flutter. It keeps typed path parameters and navigation locations generated, while giving you direct control over the `Page` object, its key, and its transition. The detail that fixed my disappearing-page bug was passing `state.pageKey` to the custom page instead of creating a new `UniqueKey` on every build.

## Why `buildPage` matters in a typed route

With a normal `GoRoute`, `pageBuilder` is the customization point. A typed route created with `go_router_builder` has the equivalent hook on `GoRouteData`:

```dart
@TypedGoRoute<InvoiceRoute>(path: '/invoices/:id')
class InvoiceRoute extends GoRouteData {
  const InvoiceRoute({required this.id});

  final String id;

  @override
  Widget build(BuildContext context, GoRouterState state) {
    return InvoiceScreen(invoiceId: id);
  }

  @override
  Page<void> buildPage(
    BuildContext context,
    GoRouterState state,
  ) {
    return CustomTransitionPage<void>(
      key: state.pageKey,
      child: InvoiceScreen(invoiceId: id),
      transitionsBuilder: (
        context,
        animation,
        secondaryAnimation,
        child,
      ) {
        return FadeTransition(
          opacity: animation,
          child: child,
        );
      },
    );
  }
}
```

The generated route still owns `/invoices/:id` and converts the URL value into the typed `id` field. `buildPage` only changes how the destination becomes a `Page`. That separation is useful because navigation safety and visual behavior stop competing inside one large hand-written route table.

| Concern | Typed route owns it | `buildPage` owns it |
| --- | --- | --- |
| URL pattern | `/invoices/:id` | — |
| `id` parsing and location generation | `InvoiceRoute` / generated code | — |
| Page key | generated `GoRouterState.pageKey` | pass it to the page |
| Transition and fullscreen behavior | — | returned `Page` |
| Screen content | `build` or `buildPage` | `child` |

## Keep the generated page key

Flutter compares pages by key when it updates a `Navigator` pages list. A route that represents `/invoices/10` should not accidentally become the same page as `/invoices/11`, and the same route should not look new on every rebuild.

This is the safe default:

```dart
return CustomTransitionPage<void>(
  key: state.pageKey,
  child: InvoiceScreen(invoiceId: id),
  transitionsBuilder: _buildTransition,
);
```

I initially used `UniqueKey()` because the transition appeared to restart reliably. It did restart reliably—but that was the problem. Any router refresh, inherited dependency update, or shell rebuild made Flutter see a brand-new page. The detail screen lost its local state, and a text field returned to its initial value.

`state.pageKey` carries the route identity that `go_router` calculated for the current match. If you need a different identity rule, make it deliberate and stable:

```dart
key: ValueKey('invoice-$id'),
```

Do not use a random key to force an animation. If a transition must restart after a meaningful state change, model that state in the widget or navigate to a genuinely different location.

## Use both animations for push and pop

The `animation` argument describes the incoming page. `secondaryAnimation` describes the page that is moving away underneath it. Ignoring the second value is fine for a simple fade, but it becomes visible when two pages slide in opposite directions.

Here is a horizontal transition that behaves sensibly for both pages:

```dart
Page<void> buildPage(
  BuildContext context,
  GoRouterState state,
) {
  return CustomTransitionPage<void>(
    key: state.pageKey,
    child: InvoiceScreen(invoiceId: id),
    transitionDuration: const Duration(milliseconds: 280),
    reverseTransitionDuration: const Duration(milliseconds: 220),
    transitionsBuilder: (
      context,
      animation,
      secondaryAnimation,
      child,
    ) {
      final incoming = Tween<Offset>(
        begin: const Offset(1, 0),
        end: Offset.zero,
      ).chain(CurveTween(curve: Curves.easeOutCubic));

      final outgoing = Tween<Offset>(
        begin: Offset.zero,
        end: const Offset(-0.20, 0),
      ).chain(CurveTween(curve: Curves.easeInCubic));

      return SlideTransition(
        position: secondaryAnimation.drive(outgoing),
        child: SlideTransition(
          position: animation.drive(incoming),
          child: child,
        ),
      );
    },
  );
}
```

The nested `SlideTransition` is intentional. The incoming route moves from the right, while the previous route shifts slightly left. During a back gesture on iOS, Flutter reverses these animations rather than calling a separate “back transition” callback.

## Avoid capturing mutable route state in the page

The typed route object is reconstructed from the location. That makes it a good value object, not a place to store screen controllers or fetched data:

```dart
class InvoiceRoute extends GoRouteData {
  const InvoiceRoute({required this.id});

  final String id;

  @override
  Page<void> buildPage(BuildContext context, GoRouterState state) {
    return CustomTransitionPage<void>(
      key: state.pageKey,
      child: InvoiceScreen(invoiceId: id),
      restorationId: 'invoice-$id',
    );
  }
}
```

Keep `TextEditingController`, loading flags, and fetched invoice data inside `InvoiceScreen` or its state-management layer. If the page is rebuilt because the router refreshes, the route remains a small description of the URL and the screen remains responsible for its own lifecycle.

There is one subtle distinction here:

- `state.pageKey` identifies a Flutter `Page` in the navigator stack.
- `restorationId` identifies restorable widget state.

They often contain the same ID, but they solve different problems. A stable page key does not automatically restore a scroll position after process death.

## A practical rule for choosing `build` or `buildPage`

Use `build` when the default `MaterialPage` or `CupertinoPage` behavior is enough. Use `buildPage` when the route itself needs a page-level decision:

| Requirement | Method |
| --- | --- |
| Render a typed screen | `build` |
| Add a fade or slide transition | `buildPage` |
| Set a restoration ID | `buildPage` |
| Use a fullscreen dialog | `buildPage` |
| Change the URL based on a condition | `redirect` |
| Load data for the screen | screen or application data layer |

Putting redirects or network calls inside `buildPage` made my route behavior hard to predict. A page builder can run more often than a developer expects, especially under nested navigators. Keep it synchronous and focused on constructing the page.

## Test page identity, not only the animation

Golden tests can verify the visual transition, but the most valuable regression test here checks that changing an unrelated dependency does not replace the page. Give the destination a stateful child with a counter or text field, trigger a router refresh, and assert that the state survives.

For a lower-level check, the important contract is simple: the returned page key must equal the router state key. I test that through a real `GoRouter` rather than constructing `GoRouterState` by hand. The state constructor has changed between package versions, while a routed widget test exercises the public behavior that matters:

```dart
testWidgets('invoice state survives a router refresh', (tester) async {
  final router = GoRouter(
    initialLocation: '/invoices/10',
    routes: [
      GoRoute(
        path: '/invoices/:id',
        builder: (context, state) =>
            InvoiceScreen(invoiceId: state.pathParameters['id']!),
      ),
    ],
  );

  await tester.pumpWidget(MaterialApp.router(routerConfig: router));
  await tester.pumpAndSettle();

  await tester.enterText(find.byType(TextField), 'draft note');
  router.refresh();
  await tester.pumpAndSettle();

  expect(find.text('draft note'), findsOneWidget);
});
```

In the production route, replace the hand-written `GoRoute` with the generated typed route. The test still checks the same failure: a random page key would recreate `InvoiceScreen` during `router.refresh()` and discard the draft.

`GoRouteData.buildPage` gives typed routes the same page-level control as hand-written `GoRoute.pageBuilder`. Keep the route value-based, preserve `state.pageKey`, use `secondaryAnimation` when the outgoing page should move, and reserve the page builder for synchronous page construction. That combination gives custom Flutter transitions without giving up generated locations or stable navigation state.
