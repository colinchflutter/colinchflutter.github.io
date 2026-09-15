---
layout: post
title: "go_router extraCodec - Preserve Complex Route Data in Flutter Web"
description: "Learn how go_router extraCodec serializes complex Flutter route data so browser refreshes and state restoration do not silently drop navigation extras."
date: 2026-09-16
tags: [go_router, navigation, FlutterWeb, Web, state_restoration]
comments: true
share: true
---

![go_router preserving route data across Flutter Web navigation](/assets/images/go-router-stateful-shell-route.png)

*The important detail is whether route data can survive a browser serialization boundary, not only whether the destination opens once.*

`go_router`'s `extra` parameter is convenient for passing an object during navigation. It becomes unreliable when that object must cross a browser refresh or state-restoration boundary. A complex object can be dropped because the browser-facing router needs a serializable representation. `extraCodec` solves that boundary by encoding the object before storage and decoding it when the route is rebuilt.

## The failure that looks like a random null

This works during an in-memory navigation:

```dart
context.go('/checkout', extra: CartSummary(
  itemCount: 3,
  totalCents: 4299,
));
```

The checkout page can read `state.extra` immediately. But a Flutter Web refresh may reconstruct the location without the original Dart object. The result is often a null value or a value with an unexpected type. The same issue matters when the app restores a route after being suspended.

| Data passed with `extra` | Refresh-safe by itself? | Better choice |
|---|---:|---|
| `String`, `int`, `bool` | Usually | Use directly or use a URL parameter |
| Small map of primitives | Usually | Add a codec when restoration matters |
| Domain object | No | Encode and decode with `extraCodec` |
| Secret or payment data | No | Keep it out of the URL and route state |

The diagram is simple: `CartSummary → encode → browser state → decode → CartSummary`. The destination should not need to know which side of that boundary it came from.

## Define a small codec

The codec must convert the domain object to a supported serializable value and reconstruct the object on the way back. Keep the wire format explicit; relying on `toString()` makes migrations and debugging painful.

```dart
import 'dart:convert';

class CartSummary {
  const CartSummary({required this.itemCount, required this.totalCents});

  final int itemCount;
  final int totalCents;

  factory CartSummary.fromJson(Map<String, dynamic> json) {
    return CartSummary(
      itemCount: json['itemCount'] as int,
      totalCents: json['totalCents'] as int,
    );
  }

  Map<String, dynamic> toJson() => {
        'itemCount': itemCount,
        'totalCents': totalCents,
      };
}

class CartSummaryCodec extends Codec<Object?, Object?> {
  const CartSummaryCodec();

  @override
  Object? decode(Object? encoded) {
    if (encoded == null) return null;
    return CartSummary.fromJson(
      Map<String, dynamic>.from(encoded as Map),
    );
  }

  @override
  Object? encode(Object? value) {
    if (value == null) return null;
    return (value as CartSummary).toJson();
  }
}
```

The nullable type is intentional. A route may have no extra data, and `decode` should treat that as a normal state rather than an exception.

## Attach it to `GoRouter`

Pass the codec once at the router boundary:

```dart
final router = GoRouter(
  extraCodec: const CartSummaryCodec(),
  routes: [
    GoRoute(
      path: '/checkout',
      builder: (context, state) {
        final summary = state.extra as CartSummary?;
        return CheckoutPage(summary: summary);
      },
    ),
  ],
);
```

Now the navigation call stays the same, while the router owns the conversion policy. Do not cast blindly if the route can also be opened from a deep link. A missing `extra` is a valid entry path, so render a reload-from-server state or redirect to a stable URL.

```dart
builder: (context, state) {
  final summary = state.extra;
  if (summary is! CartSummary) {
    return const CheckoutReloadPage();
  }
  return CheckoutPage(summary: summary);
},
```

## When `extra` is the wrong transport

`extraCodec` makes transient route data safer, but it does not turn `extra` into a public URL contract. Use path and query parameters when the destination should be bookmarkable, shareable, or reloadable without memory from the previous screen.

```dart
context.go('/orders/42?tab=receipt');
```

Use `extra` for a short-lived object such as a selected result, preview model, or return value. Never put access tokens, card details, or private customer data into a value that can be serialized by browser routing.

The practical rule is: if a refresh must recreate the screen, encode a minimal object or move the identity into the URL and load the rest from a repository. Test both `context.go(..., extra: value)` and opening the same path without `extra`; those are two different entry paths in a real Flutter Web app.

### Quick checklist

- Define a stable map-based wire format.
- Make `decode(null)` safe.
- Register `extraCodec` on the single router instance.
- Handle a missing extra for deep links and refreshes.
- Keep shareable identity in path or query parameters.
- Exclude secrets from route data.

`extraCodec` is a narrow tool, but it closes an easy-to-miss gap between successful in-app navigation and navigation that survives the browser lifecycle.
