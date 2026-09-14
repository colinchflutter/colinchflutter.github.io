---
layout: post
title: "go_router requestFocus - Control Flutter Keyboard Focus After Navigation"
description: "Learn how go_router requestFocus changes Flutter keyboard focus after route pushes, and how to avoid search and form input surprises."
date: 2026-09-15
tags: [go_router, navigation, Flutter]
comments: true
share: true
---

![go_router requestFocus controlling Flutter focus after navigation](/assets/images/go-router-stateful-shell-route.png)

`go_router` requests focus for a newly pushed route by default. That is usually helpful for accessibility, but it can be surprising when a search field loses its keyboard, a dialog steals focus, or a form jumps to a different text field after navigation. The `GoRouter.requestFocus` option gives the application one policy switch for that behavior.

## What `requestFocus` changes

The option is passed from `GoRouter` to the Navigator it manages. It controls whether a route push automatically asks the new route to take focus.

| Setting | Navigation behavior | Good fit |
| --- | --- | --- |
| `true` (default) | The new route may request focus automatically | Content pages and keyboard navigation |
| `false` | A push does not automatically move focus | Search-heavy layouts and custom focus policies |

This is not the same as disabling every `FocusNode` in the app. A page can still call `requestFocus()` explicitly, and a text field can still receive focus when the user taps it. The setting only changes the Navigator's automatic request during route transitions.

## Set a router-wide policy

Put the option on the stable `GoRouter` instance. Recreating the router in `build()` makes focus behavior harder to reproduce and can also reset navigation state.

```dart
final router = GoRouter(
  requestFocus: false,
  routes: [
    GoRoute(
      path: '/search',
      builder: (context, state) => const SearchPage(),
    ),
    GoRoute(
      path: '/results',
      builder: (context, state) => const ResultsPage(),
    ),
  ],
);

class App extends StatelessWidget {
  const App({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp.router(routerConfig: router);
  }
}
```

With `requestFocus: false`, moving from `/search` to `/results` does not make the results page hunt for a focusable child automatically. That is useful when the search experience intentionally keeps focus in a persistent field or when the page has its own focus timing.

## Restore focus deliberately when it matters

Turning the global option off does not mean every page should remain unfocused. It means each page can make that decision using its own lifecycle and accessibility requirements.

```dart
class ResultsPage extends StatefulWidget {
  const ResultsPage({super.key});

  @override
  State<ResultsPage> createState() => _ResultsPageState();
}

class _ResultsPageState extends State<ResultsPage> {
  final _headingFocus = FocusNode(debugLabel: 'results-heading');

  @override
  void initState() {
    super.initState();
    WidgetsBinding.instance.addPostFrameCallback((_) {
      if (mounted) _headingFocus.requestFocus();
    });
  }

  @override
  void dispose() {
    _headingFocus.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Results')),
      body: Focus(
        focusNode: _headingFocus,
        child: const Padding(
          padding: EdgeInsets.all(24),
          child: Text('Search results'),
        ),
      ),
    );
  }
}
```

The post-frame callback matters because the destination widget must be mounted before its focus request can be applied. I initially called `requestFocus()` directly from `initState`; it was easy to miss the request because the route had not entered the widget tree yet. A real screen should also give the focused element a meaningful semantic label or visible focus treatment.

## Keep search fields from fighting navigation

A common failure appears in a shell layout. The search field lives above a `StatefulShellRoute`, while the result page is pushed into one branch. With the default setting, the new route can claim focus as the branch changes. The keyboard closes or the caret disappears, even though the user expects the search field to remain active.

There are two reasonable policies:

1. Keep `requestFocus: true` and explicitly restore the search field after the route transition.
2. Set `requestFocus: false` and let the shell decide when the search field or page heading should receive focus.

The second policy is often easier when the shell owns a persistent search interaction. It also avoids adding a focus request to every route builder. However, do not use it as a blanket accessibility fix: keyboard users still need a predictable focus target after navigation.

## Testing the actual focus contract

Test both the router policy and the page-level exception. A test that only checks the URL can pass while the keyboard focus behavior is wrong.

```dart
testWidgets('results page chooses focus explicitly', (tester) async {
  final heading = FocusNode();
  final router = GoRouter(
    requestFocus: false,
    routes: [
      GoRoute(
        path: '/results',
        builder: (context, state) => Focus(
          focusNode: heading,
          autofocus: true,
          child: const Text('Results'),
        ),
      ),
    ],
    initialLocation: '/results',
  );

  await tester.pumpWidget(MaterialApp.router(routerConfig: router));
  await tester.pump();

  expect(heading.hasFocus, isTrue);
  heading.dispose();
});
```

Avoid asserting that a route always has focus immediately after `go()`. Focus changes are scheduled alongside the next frame, and a page may intentionally defer its request until data or an animation is ready. Assert the behavior your product promises: for example, that a search field keeps focus, or that a results heading becomes focusable after the transition.

`requestFocus` is a small `go_router` option, but it defines an important boundary. Keep the global default when normal route focus is useful, disable it when a shell or search flow owns focus, and make exceptional pages request focus explicitly after they are mounted. That separation prevents route navigation and keyboard state from competing with each other.

References: [GoRouter API](https://pub.dev/documentation/go_router/latest/go_router/GoRouter-class.html), [GoRouterDelegate API](https://pub.dev/documentation/go_router/latest/go_router/GoRouterDelegate-class.html), and [Flutter focus documentation](https://docs.flutter.dev/ui/interactivity/focus).
