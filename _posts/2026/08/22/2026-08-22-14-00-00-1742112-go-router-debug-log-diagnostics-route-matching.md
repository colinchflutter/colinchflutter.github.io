---
layout: post
title: "go_router debugLogDiagnostics - Trace Flutter Route Matching and Redirects"
description: "Learn how go_router debugLogDiagnostics exposes Flutter route matching, redirects, and navigation failures without guessing from the UI."
date: 2026-08-22
tags: [go_router, navigation, debugging, Flutter]
comments: true
share: true
---

![go_router route diagnostics in a Flutter navigation flow](/assets/images/go-router-stateful-shell-route.png)

When a Flutter route opens the wrong page, the useful question is not “why did this widget build?” It is “which location did `go_router` match, and which redirect changed it?” Setting `debugLogDiagnostics: true` on `GoRouter` prints that routing story to the debug console. It is a small switch, but it turns silent navigation decisions into something you can inspect.

## The problem with debugging routes from the screen

A route can look wrong for several different reasons:

| Symptom | Possible routing cause |
| --- | --- |
| `/account` opens the sign-in page | A redirect rejected the current user |
| A nested page disappears after a tab tap | The shell or branch matched a different location |
| A query filter resets | The app navigated with a new location instead of updating the existing one |
| An unknown URL shows an unexpected page | No route matched, so the error route handled it |

Looking only at the final screen hides the steps in between. A `print` inside every page builder is also misleading: a builder may not run when the redirect already changed the location, and a redirect may run more than once during a refresh.

## Enable diagnostics only in debug builds

The option belongs to the router, not to an individual `GoRoute`. Keep it enabled while developing and disable it in release builds so routine navigation does not fill production logs.

```dart
import 'package:flutter/foundation.dart';
import 'package:go_router/go_router.dart';

final router = GoRouter(
  debugLogDiagnostics: kDebugMode,
  initialLocation: '/home',
  routes: [
    GoRoute(
      path: '/home',
      builder: (context, state) => const HomePage(),
    ),
    GoRoute(
      path: '/settings',
      builder: (context, state) => const SettingsPage(),
    ),
  ],
);
```

`kDebugMode` is a good default for an application router. It is a compile-time constant, so the release build does not accidentally keep the verbose option enabled. For a local profile build, use an explicit configuration decision if you need to inspect navigation there.

The logs typically show the configured route tree and navigation events, including the location being processed and the result of matching. The exact formatting can vary between `go_router` versions, so treat the output as diagnostic evidence rather than a string format your code should parse.

## Read the log as a routing timeline

Consider a protected settings route:

```dart
final router = GoRouter(
  debugLogDiagnostics: kDebugMode,
  initialLocation: '/home',
  redirect: (context, state) {
    final signedIn = auth.isSignedIn;
    final goingToSignIn = state.matchedLocation == '/sign-in';

    if (!signedIn && !goingToSignIn) {
      return '/sign-in?from=${Uri.encodeComponent(state.uri.toString())}';
    }

    if (signedIn && goingToSignIn) {
      return '/home';
    }

    return null;
  },
  routes: [
    GoRoute(
      path: '/home',
      builder: (context, state) => const HomePage(),
    ),
    GoRoute(
      path: '/sign-in',
      builder: (context, state) => const SignInPage(),
    ),
    GoRoute(
      path: '/settings',
      builder: (context, state) => const SettingsPage(),
    ),
  ],
);
```

If an unauthenticated user requests `/settings`, do not stop reading at the final `/sign-in` screen. Check the sequence:

1. The requested location is `/settings`.
2. The redirect returns `/sign-in?...`.
3. The new location matches the sign-in route.
4. The redirect now returns `null` because the sign-in page is allowed.

That timeline distinguishes a normal guard from a redirect loop. If the log repeatedly alternates between two locations, inspect the conditions rather than adding more navigation calls inside page builders.

## Diagnostics for nested routes and shells

`debugLogDiagnostics` becomes especially useful with `ShellRoute` or `StatefulShellRoute`. The visible page can belong to two conceptual levels: a shell that provides persistent UI and a child route that provides the page content. A route that looks like `/settings/profile` may be matched through a shell first and then through the nested profile route.

When a tab keeps showing the wrong child, compare these three values in the log and in a temporary callback log:

| Value | What it tells you |
| --- | --- |
| Requested URI | What the app or browser actually asked for |
| Matched location | The route location accepted by the matcher |
| Current location | The router location after redirects complete |

Do not assume a branch index is the same thing as a URL. A tab tap may call `go('/files')`, while a deep link may enter `/files/42`. The shell can stay the same even though the child match changes.

## Pair diagnostics with a small redirect log

Router diagnostics explain the framework’s routing work. A one-line application log explains your own policy. Keep the two separate so the output remains readable.

```dart
redirect: (context, state) {
  final result = auth.isSignedIn && state.matchedLocation == '/sign-in'
      ? '/home'
      : null;

  debugPrint(
    '[auth redirect] location=${state.uri} '
    'matched=${state.matchedLocation} '
    'signedIn=${auth.isSignedIn} result=$result',
  );

  return result;
},
```

This is more useful than logging `context.go` at every button because it also captures browser refreshes, initial deep links, and redirects triggered by authentication changes. Remove or gate the custom log before shipping if the user state could reveal sensitive information.

## Common mistakes

`debugLogDiagnostics` does not fix navigation. It only exposes what the router is doing. A few mistakes still require code changes:

- **Expecting page-builder logs to describe redirects.** A redirect can prevent a builder from running entirely.
- **Reading a redirect log as a single navigation event.** Auth refreshes and nested matches can cause multiple evaluations.
- **Leaving diagnostics enabled in production.** It creates noisy logs and may expose locations or parameters.
- **Changing routes while investigating.** First capture the requested URI, matched location, and redirect result; otherwise the test itself changes the evidence.
- **Using only a physical device log.** Reproduce the same deep link in a debug build so the router configuration and diagnostic output are available.

The practical workflow is simple: reproduce one URL, copy the complete routing sequence, then compare it with the route tree. Once the first unexpected location appears, the bug is usually in the redirect condition, path nesting, or navigation call immediately before it.

## Quick checklist

- Enable `debugLogDiagnostics` with `kDebugMode`.
- Reproduce one navigation case at a time.
- Follow the requested URI through matching and redirects.
- Check `matchedLocation` separately from query parameters.
- Add one temporary application-level redirect log when policy is unclear.
- Disable verbose diagnostics in release builds.

The key idea is that `go_router` navigation is a pipeline, not a single widget transition. `debugLogDiagnostics` gives that pipeline a visible trace, which makes nested routes, auth guards, and deep-link failures much easier to reason about.
