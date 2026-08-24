---
layout: post
title: "go_router overridePlatformDefaultLocation - Control Flutter Startup Routes Safely"
description: "Learn how go_router overridePlatformDefaultLocation changes Flutter startup precedence and how to avoid breaking Web deep links with initialLocation."
date: 2026-08-24
tags: [go_router, navigation, Web, deep_linking]
comments: true
share: true
---

![go_router startup location precedence for Flutter deep links](/assets/images/go-router-stateful-shell-route.png)

`go_router` normally gives the platform's initial route precedence over `initialLocation`. Setting `overridePlatformDefaultLocation: true` changes that rule, so the router starts at the configured `initialLocation` even when the platform supplies another location. That is useful for a controlled demo or a native app with a fixed entry screen, but it can quietly break Flutter Web deep links if enabled globally.

## The startup decision is not just `initialLocation`

These two settings are easy to read as interchangeable, but they answer different questions:

| Setting | Role | Typical result |
|---|---|---|
| `initialLocation` | Fallback route | Used when no platform location is available |
| `overridePlatformDefaultLocation` | Precedence switch | Forces the fallback to win over the platform location |
| Browser URL | Platform location on Web | Preserves a shared or bookmarked deep link |

The practical default should usually remain `false`. With that value, opening `/orders/42` on the Web still reaches the order route instead of being replaced by `/home`.

## A safe baseline configuration

Keep `initialLocation` as a fallback and let the platform-provided route win when it exists.

```dart
final router = GoRouter(
  initialLocation: '/home',
  // false is the default; keep browser deep links intact.
  overridePlatformDefaultLocation: false,
  routes: [
    GoRoute(
      path: '/home',
      builder: (context, state) => const HomeScreen(),
    ),
    GoRoute(
      path: '/orders/:id',
      builder: (context, state) => OrderScreen(
        id: state.pathParameters['id']!,
      ),
    ),
  ],
);
```

When the application is launched without a location, `/home` is used. When a browser opens `/orders/42`, the URL is the stronger input. This is the behavior users expect from a URL-based router.

## When forcing `initialLocation` makes sense

There are cases where the platform route is not the state you want to honor. A kiosk application may always need to open `/display`, a documentation preview may need to start at `/preview`, and an integration test may need a deterministic first route.

```dart
final previewRouter = GoRouter(
  initialLocation: '/preview',
  overridePlatformDefaultLocation: true,
  routes: [
    GoRoute(
      path: '/preview',
      builder: (context, state) => const PreviewScreen(),
    ),
    GoRoute(
      path: '/home',
      builder: (context, state) => const HomeScreen(),
    ),
  ],
);
```

Here, the explicit startup contract wins. A platform-provided `/home` does not replace `/preview` during the initial route resolution. The important detail is that this is a startup policy, not a general navigation lock: later calls such as `context.go('/home')` still navigate normally.

## The Web deep-link trap

Suppose a user shares this URL:

```text
https://example.com/orders/42
```

If the Web app constructs a router with `initialLocation: '/home'` and `overridePlatformDefaultLocation: true`, the first route can become `/home`. The shared order URL is then ignored before the order page has a chance to parse its `id`.

The failure is especially confusing because in local testing the app may be opened at `/`, so everything appears correct. Test the actual refresh path as well:

| Test | Expected with `false` | Expected with `true` |
|---|---|---|
| Fresh app launch with no route | `/home` | `/home` |
| Browser opens `/orders/42` | `/orders/42` | `/home` |
| Browser refreshes `/orders/42` | `/orders/42` | `/home` |
| In-app `context.go('/home')` | `/home` | `/home` |

The setting changes the first two rows, not ordinary in-app navigation.

## Avoid platform checks inside the router unless policy differs

A common workaround is to branch on `kIsWeb` and maintain two router definitions. That can be valid when the product really has different startup policies, but it also creates two route trees that can drift apart. Prefer one route tree and keep the override disabled unless the app has a clear reason to discard platform input.

If a forced startup route is required only for a preview build, isolate the decision at configuration time:

```dart
final router = GoRouter(
  initialLocation: const bool.fromEnvironment('PREVIEW_MODE')
      ? '/preview'
      : '/home',
  overridePlatformDefaultLocation: const bool.fromEnvironment(
    'PREVIEW_MODE',
  ),
  routes: appRoutes,
);
```

This keeps the production Web build deep-link friendly while making the exceptional behavior explicit in the build configuration.

## Checklist before enabling the override

- Is the platform location intentionally disposable?
- Does the app need bookmarked and shared Web URLs to survive a refresh?
- Is the forced route limited to a kiosk, preview, or test build?
- Have you tested both a cold launch and a browser refresh on a nested path?
- Does the route tree still contain the path that the platform may provide?

`overridePlatformDefaultLocation` is best understood as a precedence switch. Leave it `false` for normal Flutter Web and deep-linkable apps. Turn it on only when the application explicitly owns the first route and can explain why an incoming platform location must be ignored.

References: [GoRouter API](https://pub.dev/documentation/go_router/latest/go_router/GoRouter-class.html), [go_router configuration](https://pub.dev/documentation/go_router/latest/topics/Configuration-topic.html).
