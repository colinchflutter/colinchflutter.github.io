---
layout: post
title: "go_router caseSensitive - Control Case-Sensitive Path Matching in Flutter"
description: "Learn how go_router caseSensitive changes Flutter path matching, when to accept mixed-case URLs, and how to normalize navigation safely."
date: 2026-09-15
tags: [go_router, navigation, Web, performance]
comments: true
share: true
---

![go_router caseSensitive controlling Flutter path matching](/assets/images/go-router-stateful-shell-route.png)

This image shows the practical difference between `/profile` and `/Profile`: one URL reaches the route, while the other can fail before the page is built. In go_router, `caseSensitive` is a small `GoRoute` option with a large effect on web deep links, bookmarked URLs, and server-side fallback behavior.

## Why path casing becomes a real bug

Flutter developers often test navigation with `context.go('/profile')` and assume that `/Profile` is equivalent. That assumption is risky when users enter URLs manually, when a marketing link uses a capital letter, or when another client generates a route path.

The default route definition is case-sensitive. A route declared as `/profile` should be treated as a different pattern from `/Profile`. That strict behavior protects a consistent URL contract, but it can also produce surprising 404-like screens if your product already has mixed-case links in the wild.

| Route definition | Requested location | Result with default setting |
|---|---|---|
| `/profile` | `/profile` | Matches |
| `/profile` | `/Profile` | Does not match |
| `/profile/:userId` | `/profile/Ada` | Matches the path pattern |
| `/profile/:userId` | `/Profile/Ada` | Does not match |

The important boundary is the path itself. `caseSensitive` is not a general-purpose lowercasing utility for every part of a URL, and it does not change the meaning of a query value such as `?tab=Settings`.

## The strict default

Keep the default when your application has a canonical, lowercase URL scheme. Omitting the option makes the intent clear enough for most routes:

```dart
final router = GoRouter(
  routes: [
    GoRoute(
      path: '/profile',
      builder: (context, state) => const ProfilePage(),
    ),
  ],
);
```

With this configuration, navigation calls and external links should both use `/profile`. A failed match is useful feedback: it reveals that a producer of URLs is violating the route contract instead of silently creating two spellings for one screen.

This is usually the better choice for authenticated dashboards, analytics URLs, and apps that need predictable browser history. It also keeps analytics aggregation simpler because one logical screen has one canonical path.

## Accepting mixed-case paths deliberately

If an existing product must accept both `/profile` and `/Profile`, configure that route explicitly:

```dart
final router = GoRouter(
  routes: [
    GoRoute(
      path: '/profile',
      caseSensitive: false,
      builder: (context, state) => const ProfilePage(),
    ),
  ],
);
```

This setting is most useful for compatibility. It can keep old links working while a migration moves callers toward lowercase paths. It should not be added to every route automatically: accepting multiple spellings expands the URL surface and makes it easier for different systems to produce inconsistent links.

For dynamic segments, the same decision applies to the static parts of the pattern:

```dart
GoRoute(
  path: '/team/:teamId/settings',
  caseSensitive: false,
  builder: (context, state) {
    final teamId = state.pathParameters['teamId']!;
    return TeamSettingsPage(teamId: teamId);
  },
)
```

Do not use this option as a substitute for validating `teamId`. If identifiers are case-sensitive in your backend, preserving the parameter exactly as received is safer than lowercasing the entire location.

## A safer migration pattern

The setting accepts legacy paths, but it does not by itself redirect every spelling to one canonical URL. If canonical URLs matter, normalize at the edge of navigation and keep route matching strict:

```dart
String canonicalProfileLocation(String rawLocation) {
  final uri = Uri.parse(rawLocation);
  final segments = [...uri.pathSegments];

  if (segments.length == 1 && segments.first.toLowerCase() == 'profile') {
    return uri.replace(path: '/profile').toString();
  }

  return rawLocation;
}

void openProfile(BuildContext context, String rawLocation) {
  context.go(canonicalProfileLocation(rawLocation));
}
```

For a browser-facing migration, a redirect or server rewrite may be more appropriate than changing an imperative `go()` call. The key is to choose one owner for normalization. If the app, web server, and backend all rewrite independently, debugging a deep link becomes harder.

## Common traps

### Mixing case-insensitive matching with case-sensitive IDs

`caseSensitive: false` can be reasonable for `/Team/:teamId`, but it does not mean that `ABC` and `abc` identify the same database record. Keep route matching policy separate from domain identifier policy.

### Testing only imperative navigation

`context.go('/profile')` exercises the spelling your code generated. Add tests for direct locations such as `/Profile`, browser refresh, and copied deep links when compatibility is required.

### Changing a shared route without checking redirects

Redirect logic often compares `state.matchedLocation` or `state.uri.path`. After accepting mixed-case paths, those values can preserve the incoming spelling. Compare normalized values when the redirect decision is meant to be case-insensitive.

## Practical decision checklist

| Situation | Recommended policy |
|---|---|
| New app with lowercase URLs | Keep `caseSensitive` at its default |
| Legacy links with inconsistent casing | Temporarily use `caseSensitive: false` |
| Case-sensitive resource IDs | Match paths deliberately and preserve parameters |
| SEO or analytics needs one URL | Normalize and redirect to a canonical path |
| Unknown external URL producers | Add direct-location tests before relaxing matching |

The useful rule is simple: strict matching defines a clean URL contract; case-insensitive matching is a compatibility tool. Decide per route, document the exception, and test the URL forms users can actually open.
