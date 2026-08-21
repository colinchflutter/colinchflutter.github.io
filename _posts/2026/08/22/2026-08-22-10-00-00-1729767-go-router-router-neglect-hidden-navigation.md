---
layout: post
title: "go_router routerNeglect - Navigate in Flutter Without Changing the Browser URL"
description: "Learn how go_router routerNeglect controls browser URL updates in Flutter Web, with safe patterns for internal flows and back navigation."
date: 2026-08-22
tags: [go_router, navigation, FlutterWeb, Web]
comments: true
share: true
---

![go_router controlling Flutter Web navigation and browser URL updates](/assets/images/go-router-stateful-shell-route.png)

`go_router`'s `routerNeglect` is useful when a Flutter Web interaction should change the visible screen without creating a new shareable URL. It is a good fit for an in-app preview, a temporary confirmation step, or a modal-like flow that still needs a real Flutter route. It is not a general replacement for URL-based navigation: if a user should be able to refresh, bookmark, or share the destination, the URL should change.

## The problem: every navigation becomes a URL

On the web, this code changes both the Flutter page and the address bar:

```dart
context.go('/checkout/complete');
```

That is normally exactly what we want. The browser can restore `/checkout/complete`, analytics can identify the screen, and the user can copy the link. But some transitions are only part of the current task. For example, after a file upload, showing a local completion screen at `/upload/done` can make the browser history noisy. Pressing Back then walks through a sequence of implementation details instead of meaningful pages.

`routerNeglect` addresses this specific problem. A neglected navigation still updates the router's page stack, but the platform URL is not updated for that navigation.

| Navigation | Flutter screen | Browser URL | Typical use |
| --- | --- | --- | --- |
| `context.go('/details')` | Changes | Changes | Shareable page |
| `router.neglect(() => context.go('/details'))` | Changes | Stays | Temporary internal flow |
| `routerNeglect: true` | Changes | Usually stays | App-wide policy |

The key decision is not whether a screen looks temporary. It is whether its location is part of the app's public address space.

## Use `GoRouter.neglect` for one transition

The scoped API is safer for most applications because it makes the exception visible at the call site. Keep normal navigation normal, and wrap only the transition that should not write a URL:

```dart
class UploadCompleteButton extends StatelessWidget {
  const UploadCompleteButton({super.key});

  @override
  Widget build(BuildContext context) {
    final router = GoRouter.of(context);

    return FilledButton(
      onPressed: () {
        router.neglect(() {
          context.go('/upload/done');
        });
      },
      child: const Text('Show completion'),
    );
  }
}
```

The callback performs ordinary `go_router` navigation. `neglect` changes how that navigation is reported to the platform; it does not turn the route into a widget overlay and it does not prevent the destination from being built.

There is a small trap here: do not hide all navigation inside a helper named `go`. A helper that sometimes neglects the URL makes route behavior difficult to infer from a button. Prefer an explicit method name when the distinction matters:

```dart
void openTemporaryResult(BuildContext context) {
  GoRouter.of(context).neglect(() {
    context.go('/upload/done');
  });
}
```

## Use the constructor flag only for a deliberate global rule

`GoRouter` also exposes a `routerNeglect` constructor option. It applies the policy to navigation initiated through that router:

```dart
final router = GoRouter(
  routerNeglect: true,
  routes: [
    GoRoute(
      path: '/',
      builder: (context, state) => const HomePage(),
    ),
    GoRoute(
      path: '/settings',
      builder: (context, state) => const SettingsPage(),
    ),
  ],
);
```

This can make sense for an embedded Flutter Web experience where the host page owns the URL. It is usually a poor default for a normal website. A global setting means `/settings` may render correctly but fail to become a usable browser location. A later deep link, refresh, or analytics report then has less information than the screen itself suggests.

If only one part of the product is embedded, keep the main router URL-aware and use the scoped API inside that feature. The smaller scope also makes it easier to test the intended behavior.

## Back navigation is still a product decision

Neglecting the URL does not automatically make browser history and Flutter history identical. The route can still be present in the app's navigation state, while the address bar remains on the previous location. That is why a temporary route should have an intentional exit action:

```dart
class UploadDonePage extends StatelessWidget {
  const UploadDonePage({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Upload complete')),
      body: Center(
        child: FilledButton(
          onPressed: () => context.go('/files'),
          child: const Text('Return to files'),
        ),
      ),
    );
  }
}
```

For a flow that behaves like a dialog, a `DialogRoute` or `showModalBottomSheet` may be clearer. Use `routerNeglect` when the destination deserves route-level lifecycle, restoration, or a dedicated page, but should not become a public URL.

## Test the browser behavior, not only the widget tree

A widget test can confirm that the destination appears, but it will not prove that the browser location stayed unchanged. On Flutter Web, verify both values around the action:

```dart
testWidgets('temporary result keeps the public location', (tester) async {
  final router = GoRouter(
    initialLocation: '/files',
    routes: [
      GoRoute(path: '/files', builder: (_, __) => const FilesPage()),
      GoRoute(path: '/upload/done', builder: (_, __) => const UploadDonePage()),
    ],
  );

  await tester.pumpWidget(
    MaterialApp.router(routerConfig: router),
  );
  router.neglect(() {
    router.go('/upload/done');
  });
  await tester.pumpAndSettle();

  expect(find.text('Upload complete'), findsOneWidget);
  expect(router.routeInformationProvider.value.uri.path, '/files');
});
```

The exact browser integration can vary with the test environment, so an end-to-end check is still valuable. Test a refresh and a copied URL too. If a user can land directly on the temporary route and the screen cannot reconstruct its data, it probably belongs in a URL-aware route instead.

## Practical rule

Use ordinary `go_router` navigation for destinations that represent application state. Use `GoRouter.neglect` for a narrowly scoped, temporary transition. Set `routerNeglect: true` only when the entire router intentionally lives inside a host that owns the address bar.

That separation keeps the browser URL honest while still giving Flutter enough routing structure for complex internal flows.
