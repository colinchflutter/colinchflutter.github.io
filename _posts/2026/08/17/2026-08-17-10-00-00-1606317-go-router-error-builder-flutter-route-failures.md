---
layout: post
title: "go_router errorBuilder - Handle Flutter Route Failures with a Custom Error Screen"
description: "Learn how to use go_router errorBuilder to handle Flutter 404 routes, failed redirects, and invalid deep links with a useful recovery UI."
date: 2026-08-17
tags: [go_router, navigation, testing, Web]
comments: true
share: true
---

![Flutter route error screen showing a failed deep link and a recovery action](https://images.unsplash.com/photo-1551650975-87deedd944c3?w=1200&q=80)

When a Flutter app uses `go_router`, an invalid deep link should end at a deliberate error screen, not a blank page or an exception-looking debug message. The router-level `errorBuilder` is the right place to turn unknown paths, failed redirects, and malformed route data into a recoverable UI.

## Why the default error page is not enough

The failure can happen before the destination widget is built. That means putting a `try/catch` inside `ProductPage` will not handle an unknown `/products/999` path, and a widget-level loading state cannot explain why `/settings/profile` failed to match.

There are three details worth keeping visible:

| Value | Why it matters |
| --- | --- |
| `state.uri` | Shows the path the user actually opened, including query parameters |
| `state.error` | Gives a diagnostic description for logs and debugging |
| recovery action | Moves the user to a known route instead of repeating the failure |

The error screen should expose the first value to the user only when it is safe, while the second value is usually better suited to logs.

## Add a router-level errorBuilder

The following router handles ordinary pages and sends every routing failure to one small, reusable screen.

```dart
final router = GoRouter(
  initialLocation: '/',
  routes: [
    GoRoute(
      path: '/',
      builder: (context, state) => const HomePage(),
    ),
    GoRoute(
      path: '/products/:id',
      builder: (context, state) {
        final id = state.pathParameters['id'];
        if (id == null || int.tryParse(id) == null) {
          throw GoException('The product id is invalid.');
        }
        return ProductPage(productId: int.parse(id));
      },
    ),
  ],
  errorBuilder: (context, state) {
    return RouteErrorPage(
      location: state.uri.toString(),
      error: state.error,
    );
  },
);
```

The `errorBuilder` belongs on `GoRouter`, so it can cover both route matching failures and exceptions thrown while a matched route is being built. Keeping the builder at this level also prevents every feature module from inventing a slightly different 404 page.

Here is a compact recovery screen. `context.go('/')` is intentional: retrying the same invalid URL often produces the same error and gives the user no way out.

```dart
class RouteErrorPage extends StatelessWidget {
  const RouteErrorPage({
    required this.location,
    required this.error,
    super.key,
  });

  final String location;
  final Exception? error;

  @override
  Widget build(BuildContext context) {
    final isNotFound = error == null;

    return Scaffold(
      appBar: AppBar(title: const Text('Page unavailable')),
      body: Center(
        child: Padding(
          padding: const EdgeInsets.all(24),
          child: Column(
            mainAxisSize: MainAxisSize.min,
            children: [
              Text(
                isNotFound ? '404' : 'Something went wrong',
                style: Theme.of(context).textTheme.headlineMedium,
              ),
              const SizedBox(height: 12),
              Text('We could not open $location.'),
              const SizedBox(height: 20),
              FilledButton(
                onPressed: () => context.go('/'),
                child: const Text('Go to home'),
              ),
            ],
          ),
        ),
      ),
    );
  }
}
```

## Separate user messaging from diagnostics

Showing `state.error.toString()` directly is useful during development, but it can leak backend details, internal route names, or tokens embedded in a URL. I keep the public copy stable and send the technical data to the app's logging layer instead.

```dart
errorBuilder: (context, state) {
  debugPrint(
    'go_router failure: ${state.uri} - ${state.error}',
  );

  return RouteErrorPage(
    location: state.uri.path,
    error: state.error,
  );
},
```

On web, `state.uri.path` is usually a better display value than `state.uri.toString()` because query parameters can contain search terms or identifiers. On mobile, keep the full URI in private logs when reproducing a deep-link issue.

## Common traps

The most confusing mistake is placing `errorBuilder` on a `GoRoute` and expecting it to handle every router failure. Route-level error pages are useful for a local branch, but a top-level handler gives unknown paths and redirect failures a consistent fallback.

Another trap is calling `context.pop()` from the error page. A browser deep link can open with an empty navigation stack, and a cold-start mobile link may have nowhere to pop to. A known route such as `/` is safer.

Test at least these cases:

- `/does-not-exist` renders the error page.
- `/products/not-a-number` does not build `ProductPage`.
- A failed auth redirect still exposes a route the user can reach.
- The home button works when the app was launched directly from a deep link.

`errorBuilder` is a small API surface, but it closes a major navigation gap. Match failures become observable, users get a clear recovery path, and the router remains the single place that owns route-level error behavior.
