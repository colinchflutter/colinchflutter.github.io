---
layout: post
title: "go_router requestFocus - Control Keyboard Focus After Flutter Navigation"
description: "Learn how go_router requestFocus changes keyboard focus after Flutter route pushes, and how to combine it with autofocus and FocusNode safely."
date: 2026-08-25
tags: [go_router, navigation, Flutter, accessibility, testing]
comments: true
share: true
---

![go_router controlling keyboard focus during Flutter navigation](/assets/images/go-router-stateful-shell-route.png)

`go_router` requests focus for the new top route by default. That is usually helpful for keyboard and accessibility users, but it can also steal focus from a search field, dismiss an intentional text cursor, or make a Flutter Web form feel different after every `push`. The `requestFocus` option gives the router a single place to change that behavior.

## What `requestFocus` actually controls

The option belongs to `GoRouter`, not to an individual `GoRoute`:

```dart
final router = GoRouter(
  requestFocus: false,
  routes: [
    GoRoute(
      path: '/',
      builder: (context, state) => const HomeScreen(),
    ),
    GoRoute(
      path: '/search',
      builder: (context, state) => const SearchScreen(),
    ),
  ],
);
```

With the default `requestFocus: true`, the Navigator created by the router may request focus when a new top route is pushed. With `false`, navigation does not automatically move focus to the new route. The route still changes, its widgets still build, and a widget can still request focus explicitly.

That last distinction matters. `requestFocus` is not a replacement for `autofocus`, and it does not disable every focus request in the application.

| Setting | What it affects | What it does not affect |
| --- | --- | --- |
| `requestFocus` | Router-driven focus after a new route is pushed | Explicit `FocusNode.requestFocus()` calls |
| `autofocus: true` | A particular focusable widget | Focus behavior on other routes |
| `FocusScope.of(context).requestFocus()` | A deliberate focus change | Future route pushes |

## A practical form flow

Consider a search page that opens a filter page. The user types a query, taps a filter button, changes one option, and returns. Moving focus automatically on the filter route is not useful, and on desktop it can produce an unexpected keyboard highlight.

Keep the router quiet and assign focus where the screen has a real reason to do so:

```dart
final router = GoRouter(
  requestFocus: false,
  routes: [
    GoRoute(
      path: '/search',
      builder: (context, state) => const SearchScreen(),
      routes: [
        GoRoute(
          path: 'filters',
          builder: (context, state) => const FilterScreen(),
        ),
      ],
    ),
  ],
);

class SearchScreen extends StatefulWidget {
  const SearchScreen({super.key});

  @override
  State<SearchScreen> createState() => _SearchScreenState();
}

class _SearchScreenState extends State<SearchScreen> {
  final queryFocusNode = FocusNode();

  @override
  void dispose() {
    queryFocusNode.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return TextField(
      focusNode: queryFocusNode,
      decoration: const InputDecoration(labelText: 'Search'),
    );
  }
}
```

If the search field should receive focus only on the initial visit, do that explicitly rather than relying on the router:

```dart
@override
void initState() {
  super.initState();
  WidgetsBinding.instance.addPostFrameCallback((_) {
    if (mounted) queryFocusNode.requestFocus();
  });
}
```

This makes the intent visible: the search screen owns its initial focus, while navigation does not make a decision for every screen in the app.

## `go()` and `push()` are still different

`requestFocus` does not change the navigation model. `context.go('/search/filters')` replaces the displayed route configuration, while `context.push('/search/filters')` adds a page to the Navigator stack. Both can create a new top route, so both can be affected by the router's focus policy.

For a temporary filter screen, `push` is often the better match because the user expects the search screen to remain underneath. For a URL that represents the current application state, `go` is usually clearer. Focus policy should not be used to compensate for choosing the wrong navigation method.

## Stateful shells and nested navigators

`StatefulShellRoute` adds another boundary. Each branch can have its own Navigator, but `requestFocus` is configured on the `GoRouter`. Test the actual shell structure if a tab switch appears to focus or blur a widget unexpectedly. A focus policy that feels correct on a single Navigator may feel too aggressive when tabs preserve their own stacks.

I use this rule of thumb:

- Use `requestFocus: true` when most destinations are document-like screens and keyboard users should land on the new route.
- Use `requestFocus: false` when the app contains search, editors, media controls, or persistent desktop shortcuts.
- Add explicit focus requests to the few screens that need them.
- Do not use `autofocus` on every route as a workaround; it can compete with the user's current focus.

## Testing the behavior

A widget test should verify the focus contract, not just that navigation reaches the right page:

```dart
testWidgets('keeps focus policy explicit after navigation', (tester) async {
  await tester.pumpWidget(MaterialApp.router(routerConfig: router));

  await tester.tap(find.byKey(const ValueKey('open-filters')));
  await tester.pumpAndSettle();

  expect(find.byType(FilterScreen), findsOneWidget);
  // Add an explicit focus assertion when FilterScreen owns a FocusNode.
});
```

For a real assertion, expose a key on the intended field and check `hasFocus` after the route settles. Also test Android back, browser back, and a keyboard-only flow on Flutter Web. The bug usually appears at one of those boundaries rather than on the initial tap.

`requestFocus` is a small option with app-wide consequences. Set it deliberately, then let each route document its own focus needs. That produces more predictable navigation for touch users, keyboard users, and anyone returning to a form with unfinished input.

References: [GoRouter API](https://pub.dev/documentation/go_router/latest/go_router/GoRouter-class.html) and [RouteBuilder API](https://pub.dev/documentation/go_router/latest/go_router/RouteBuilder-class.html).
