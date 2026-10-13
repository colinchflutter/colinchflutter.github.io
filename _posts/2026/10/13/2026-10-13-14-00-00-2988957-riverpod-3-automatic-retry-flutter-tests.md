---
layout: post
title: "Riverpod 3 Automatic Retry - Stop Flutter Tests and One-Shot Providers from Repeating Work"
description: "Riverpod 3 automatically retries failed providers, which can duplicate one-shot work and make Flutter tests fail. Classify errors, bound retries, and configure ProviderScope safely."
date: 2026-10-13
tags: [riverpod, testing, networking, migration, performance, Flutter]
comments: true
share: true
---

![Riverpod 3 automatic retry flow for Flutter providers and tests](https://images.unsplash.com/photo-1555066931-4365d14bab8c?auto=format&fit=crop&w=1600&q=80)

The safe choice is to keep Riverpod 3 automatic retry for transient reads such as a temporarily unavailable API, but disable or bound it for tests and one-shot side effects. It does not apply to every failed provider: a validation error, a missing permission, or a malformed response will not become valid just because the request runs again.

## The migration trap is behavioral, not just a compile error

Riverpod 3 retries a provider that fails during initialization until it succeeds or is disposed. The official default starts after 200 ms, doubles the delay after each failure, and stops at a 6.4-second delay. That is useful for a flaky network, but it changes the meaning of “the provider failed.” A test that expects one request can now observe several attempts, and a provider that performs a non-idempotent operation can repeat work.

| Provider job | Default retry fit | Safer policy |
| --- | --- | --- |
| Read a product list | Usually useful | Retry network timeouts, cap attempts |
| Decode a server response | Poor | Return the error immediately |
| Submit payment or create an order | Dangerous | Keep the mutation outside a read provider |
| Widget test of an error state | Noisy | Disable retry in the test container |

The gap in many migration notes is that changing `AutoDisposeNotifier` to `Notifier` is mechanical, while retry changes timing and request count. The official migration guide documents both the new default and the `retry` hook, but the choice still belongs to the application boundary.

## Give retry a finite, error-aware policy

This policy retries only an explicitly transient failure and gives the provider at most three attempts. The exception types are application examples; map them to the errors produced by your HTTP client.

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';

class TemporaryNetworkException implements Exception {
  const TemporaryNetworkException(this.message);
  final String message;
}

class InvalidPayloadException implements Exception {
  const InvalidPayloadException();
}

Duration? appRetry(int retryCount, Object error) {
  // Permanent failures should reach AsyncError immediately.
  if (error is InvalidPayloadException) return null;
  if (error is! TemporaryNetworkException) return null;

  // retryCount is zero-based. This permits the initial call plus 3 retries.
  if (retryCount >= 3) return null;
  return Duration(milliseconds: 200 << retryCount);
}

final catalogProvider = FutureProvider<List<String>>(
  (ref) async {
    final response = await catalogApi.fetch();
    if (!response.isValid) {
      throw const InvalidPayloadException();
    }
    return response.items;
  },
  retry: appRetry,
);

void main() {
  runApp(
    ProviderScope(
      retry: appRetry,
      child: const App(),
    ),
  );
}
```

The global policy is a useful default, but a provider can override it. Keep the retry decision next to the provider when its error semantics are special. For example, a session provider may stop on `401`, while a catalog read can retry a timeout. Do not treat every `DioException` as transient without checking status and request type.

## Keep one-shot work out of an automatically retried read

An order creation call, analytics event, or file upload is not automatically safe to repeat. A provider rebuild or retry can send the same command twice. Use an idempotency key at the API boundary, or model the action as a user-triggered mutation with its own pending state. A retry policy is not a substitute for server-side idempotency.

```dart
final orderProvider = FutureProvider<Order>((ref) async {
  // This is a read of existing state, not the place to create an order.
  return ordersApi.currentOrder();
});

Future<Order> submitOrder(Cart cart) {
  final key = createStableIdempotencyKeyForThisSubmission();
  return ordersApi.createOrder(cart, idempotencyKey: key);
}
```

The stable key must belong to one user action. Generating a new key inside a retried provider defeats the purpose because each attempt looks like a new order to the server.

## Disable retry in tests that assert a terminal error

A failing provider is often intentionally permanent in a unit or widget test. Configure the test scope rather than changing the production policy. `ProviderContainer.test` also disposes the container at the end of the test, which makes the lifecycle explicit.

```dart
test('invalid payload is exposed without another API call', () async {
  final container = ProviderContainer.test(
    retry: (_, __) => null,
    overrides: [
      catalogApiProvider.overrideWithValue(FakeCatalogApi.invalid()),
    ],
  );

  await expectLater(
    container.read(catalogProvider.future),
    throwsA(isA<ProviderException>()),
  );
});
```

For widget tests, the equivalent boundary is `ProviderScope(retry: (_, __) => null, child: ...)`. Avoid using a long `pumpAndSettle` duration to “wait out” retries. It can advance through multiple scheduled attempts and surface several exceptions in the same test. Assert the state you need, or inject a fake API that completes deterministically.

The Riverpod project has a discussion reproducing this behavior: a test without a long duration appeared to pass, while a 30-second `pumpAndSettle` allowed retries to produce additional exceptions. That is a test configuration signal, not proof that the widget correctly handled the failure.

## Release checklist

- Search providers that perform POST, PUT, DELETE, uploads, payments, or analytics from `build` or a provider initializer.
- For each failing provider, classify timeout, authorization, validation, decoding, and server errors separately.
- Set a finite retry count for network reads and return `null` for permanent errors.
- Disable retry in tests that verify one terminal error or one API invocation.
- Add a call-count assertion to tests around retries.
- Use an idempotency key for commands that may be repeated after a timeout.
- Re-check `AsyncError` UI and logging after upgrading; a provider can remain loading while retry is scheduled.

Riverpod 3 retry is a useful resilience feature when the operation is a repeatable read. The migration is incomplete if the code compiles but a request count, test timing, or command safety assumption has changed. Bound the policy at the provider boundary, and make test containers deterministic.

Sources: [Riverpod 3 migration guide](https://riverpod.dev/docs/3.0_migration), [Riverpod 3 automatic retry documentation](https://riverpod.dev/docs/whats_new#automatic-retry), and [Riverpod discussion reproducing retry-related Flutter test failures](https://github.com/rrousselGit/riverpod/discussions/4431).
