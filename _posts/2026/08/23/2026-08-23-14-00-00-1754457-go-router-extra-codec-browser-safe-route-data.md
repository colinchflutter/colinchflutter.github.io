---
layout: post
title: "go_router extraCodec - Preserve Complex Flutter Route Data Across Browser Restores"
description: "Learn how go_router extraCodec serializes complex Flutter route data for browser history, refreshes, and deep-link restoration."
date: 2026-08-23
tags: [go_router, navigation, FlutterWeb, Web, testing]
comments: true
share: true
---

![go_router extraCodec serializing complex Flutter route data for browser restoration](/assets/images/go-router-extra-typed-route-data.png)

`go_router`'s `extra` is convenient for passing an object to the next Flutter screen, but it has a sharp edge on the web: browser history and refreshes need serializable route state. `extraCodec` solves that boundary by defining how a custom object becomes JSON-compatible data and how it is rebuilt later. It is useful when a route needs temporary structured data, but it is not a replacement for putting shareable state in the URL.

## Why ordinary extra disappears

Passing a small value works as expected:

```dart
context.go('/checkout', extra: cart);
```

The destination can read it from `GoRouterState`:

```dart
final cart = state.extra as Cart;
```

This is fine while the same in-memory router session owns the navigation. The problem appears when the route match list is serialized for browser history or reconstructed after a refresh. A custom Dart object such as `Cart` has no automatic JSON representation. Without a codec, the value can be dropped and the checkout page may receive `null` after restoration.

The failure is easy to miss because a normal button tap works. I only noticed it after opening the same route with browser back and then refreshing the page. That is the important test: do not validate `extra` only through the forward navigation that created it.

## Create a serializable route object

Keep the object small and explicit. The codec should not try to serialize an entire repository model with caches, controllers, or service references.

```dart
class CheckoutDraft {
  const CheckoutDraft({
    required this.productId,
    required this.quantity,
    required this.coupon,
  });

  final String productId;
  final int quantity;
  final String? coupon;
}
```

Now define a codec that converts the object to a JSON-compatible map and decodes it back. The `GoRouterState.extra` value is typed as `Object?`, so the decoder also needs to reject unexpected data instead of silently constructing a broken draft.

```dart
class CheckoutDraftCodec extends Codec<Object?, Object?> {
  const CheckoutDraftCodec();

  @override
  Converter<Object?, Object?> get encoder =>
      const _CheckoutDraftEncoder();

  @override
  Converter<Object?, Object?> get decoder =>
      const _CheckoutDraftDecoder();
}

class _CheckoutDraftEncoder
    extends Converter<Object?, Object?> {
  const _CheckoutDraftEncoder();

  @override
  Object? convert(Object? value) {
    if (value == null) {
      return null;
    }
    if (value is! CheckoutDraft) {
      throw const FormatException('Unsupported extra value');
    }
    return {
      'productId': value.productId,
      'quantity': value.quantity,
      'coupon': value.coupon,
    };
  }
}

class _CheckoutDraftDecoder
    extends Converter<Object?, Object?> {
  const _CheckoutDraftDecoder();

  @override
  Object? convert(Object? value) {
    if (value == null) {
      return null;
    }
    if (value is! Map) {
      throw const FormatException('Invalid checkout draft');
    }

    final productId = value['productId'];
    final quantity = value['quantity'];
    final coupon = value['coupon'];

    if (productId is! String || quantity is! int) {
      throw const FormatException('Incomplete checkout draft');
    }

    return CheckoutDraft(
      productId: productId,
      quantity: quantity,
      coupon: coupon is String ? coupon : null,
    );
  }
}
```

`Codec` is from `dart:convert`, so add the import before configuring the router:

```dart
import 'dart:convert';
import 'package:go_router/go_router.dart';
```

## Register extraCodec on GoRouter

Pass the codec to the router itself. It applies to route `extra` values that go through this router, including values sent with `context.go`, `context.push`, and their named equivalents.

```dart
final router = GoRouter(
  extraCodec: const CheckoutDraftCodec(),
  routes: [
    GoRoute(
      path: '/checkout',
      builder: (context, state) {
        final draft = state.extra;

        if (draft is! CheckoutDraft) {
          return const InvalidCheckoutScreen();
        }

        return CheckoutScreen(draft: draft);
      },
    ),
  ],
);
```

The navigation call stays simple:

```dart
context.go(
  '/checkout',
  extra: CheckoutDraft(
    productId: 'sku-42',
    quantity: 2,
    coupon: 'SPRING10',
  ),
);
```

The codec makes the route state restorable, but it does not make the route URL descriptive. A user who copies `/checkout` still has no product ID in the address bar. If the destination must be bookmarked or shared, a path parameter or query parameter remains the better contract. I use `extraCodec` for short-lived workflow state and URL parameters for public navigation state; this boundary also complements [go_router namedLocation]({% post_url 2026-08-21-10-00-00-1705077-go-router-named-location-safe-navigation %}) when constructing shareable locations.

## Test the failure cases, not only the happy path

The most valuable test starts with a real route state and verifies that encoding followed by decoding preserves the fields that matter.

```dart
test('round trips a checkout draft', () {
  const codec = CheckoutDraftCodec();
  const original = CheckoutDraft(
    productId: 'sku-42',
    quantity: 2,
    coupon: 'SPRING10',
  );

  final encoded = codec.encode(original);
  final restored = codec.decode(encoded) as CheckoutDraft;

  expect(restored.productId, 'sku-42');
  expect(restored.quantity, 2);
  expect(restored.coupon, 'SPRING10');
});
```

Also test `null` and malformed maps. A browser history entry can outlive a model version, so decoders should fail predictably and the route builder should show a recovery screen or redirect to a safe location. Do not put secrets, access tokens, or large data blobs in `extra`; browser persistence is the wrong place for them.

| Route data | Best location | Reason |
| --- | --- | --- |
| Product ID or filter | Path/query parameter | Shareable and inspectable URL |
| Small temporary object | `extraCodec` | Structured workflow state with restoration support |
| Auth token or large model | Repository/storage | Security, size, and lifecycle control |

The practical checklist is short: make the extra object deliberately serializable, register one codec on the router, validate decoded input, and test browser refresh separately from button navigation. `extraCodec` closes the gap between an in-memory Flutter route and a route state that must survive the browser, while leaving the URL responsible for the state users should be able to share.

Reference: [go_router Navigation topic](https://pub.dev/documentation/go_router/latest/topics/Navigation-topic.html).
