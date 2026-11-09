---
layout: post
title: "package:http RetryClient - Retry Flutter Requests Without Duplicating Uploads"
description: "Use Dart package:http RetryClient safely in Flutter by separating idempotent reads from streamed uploads, bounding retries, and preserving an explicit recovery path for POST requests."
date: 2026-11-09
tags: [http, networking, performance, testing, Flutter, Web]
comments: true
share: true
---

![package:http RetryClient request retry flow for Flutter](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?auto=format&fit=crop&w=1600&q=80)

*The diagram to keep in mind is the boundary between a repeatable request and a request that may create a second server-side effect.*

If a Flutter app mainly fetches JSON, `package:http`'s `RetryClient` is a useful small wrapper. If the same client sends large `StreamedRequest` bodies or creates orders with `POST`, wrapping every request is the wrong fix: `RetryClient` copies request data so it can resend it, and a repeated write can become a duplicate operation. The production choice is to retry bounded, idempotent reads and give uploads or writes their own policy.

## The failure that looks like a network fix

`RetryClient` is attractive because a temporary `503` or connection error disappears without adding retry code to every repository. The trap is that the transport layer does not know whether a request is safe to repeat.

| Request | Automatic retry decision | Reason |
| --- | --- | --- |
| `GET /profile` | Usually yes, with a small limit | Re-reading does not create a new resource |
| `PUT /settings` | Only with an idempotent server contract | The same state can be written twice safely only by agreement |
| `POST /orders` | No by default | A timeout does not prove the server did nothing |
| Large `StreamedRequest` upload | Prefer no transparent retry | The body is copied for resending and can raise memory use |

The official `http` API documents that `RetryClient` copies request data to support resending. That is a useful guarantee for a small request body, not a promise that an arbitrary stream is cheap or semantically safe to repeat.

## Keep the retry policy narrow

The following client retries only requests whose method is safe to repeat. It also limits the response statuses instead of treating every server error as temporary.

```dart
import 'package:http/http.dart' as http;
import 'package:http/retry.dart';

http.Client buildReadClient() {
  final base = http.Client();
  return RetryClient(
    base,
    retries: 2,
    when: (response) {
      final method = response.request?.method.toUpperCase();
      final retryableMethod = method == 'GET' || method == 'HEAD';
      final retryableStatus = {
        408, 429, 500, 502, 503, 504,
      }.contains(response.statusCode);
      return retryableMethod && retryableStatus;
    },
    delay: (retryCount) => Duration(milliseconds: 250 * retryCount),
  );
}
```

Use this client for repositories that issue reads, and close it with the owning service. A retry count of two means the original request plus at most two retries. The delay above is deliberately short and deterministic for a small example; a real service should add jitter so many devices do not retry at the same instant.

Do not share this wrapper with a file-upload repository just because both repositories use HTTP. A `StreamedRequest` may contain a file-sized body, and the retry implementation's copy is a memory decision as well as a network decision.

## Give writes an explicit recovery contract

For a write, the first question is not “how many retries?” It is “can the server identify the same operation if the client does not receive the response?” If the answer is no, a timeout leaves an ambiguous result. Retrying blindly can create two orders, two payments, or two upload records.

For APIs that support idempotency keys, create the key before the first attempt and send the same key on every deliberate retry. Keep the key tied to the user action, not to a widget rebuild.

```dart
import 'package:http/http.dart' as http;

Future<http.Response> createOrder({
  required http.Client client,
  required String operationId,
  required String jsonBody,
}) {
  return client.post(
    Uri.parse('https://api.example.test/orders'),
    headers: {
      'content-type': 'application/json',
      'idempotency-key': operationId,
    },
    body: jsonBody,
  );
}
```

The header name is an example contract, not a feature supplied by `package:http`. The backend must persist the key and return the same result for a repeated key. Without server support, show an unresolved state and query the operation later, or let the user retry only after the product can explain the duplicate risk.

## Test the edge cases instead of only the happy path

A mock client can verify the policy without waiting for real network failures. The important cases are different: a retryable read, a permanent response, a write that must not be repeated, and a body large enough that a transparent copy is unacceptable.

```dart
test('does not retry POST when the server returns 503', () async {
  var calls = 0;
  final inner = _CountingClient((request) {
    calls++;
    return http.Response('busy', 503, request: request);
  });
  final client = RetryClient(inner, retries: 2);

  final response = await client.post(
    Uri.parse('https://api.example.test/orders'),
    body: '{"total": 10}',
  );

  expect(response.statusCode, 503);
  expect(calls, 1);
  client.close();
});
```

The exact fake client is project test infrastructure, so the snippet focuses on the assertion that matters: a failed `POST` is not silently multiplied by a transport helper. Add a separate test for a `GET` that returns `503`, then `200`, and assert two calls. For an upload, measure the chosen body representation under the largest supported file size; a clean small-file test does not prove that copying is affordable.

## Checklist before shipping

- [ ] The retrying client is scoped to idempotent reads, not every repository.
- [ ] `POST` and streamed uploads have a server-side idempotency or resume contract.
- [ ] Retryable statuses and the retry count are explicit.
- [ ] Delays include jitter in the production implementation.
- [ ] Clients are closed and not recreated on every widget build.
- [ ] Tests cover timeout-after-server-acceptance as an ambiguous result.
- [ ] The largest upload is tested without relying on a copied in-memory body.

`RetryClient` is a transport convenience, not a transaction manager. Use it where repeating the request is cheap and semantically safe. For writes and large streams, make the operation recoverable at the API boundary first; only then decide whether a client retry belongs in the flow.

Sources checked 2026-11-09: [RetryClient API](https://pub.dev/documentation/http/latest/retry/RetryClient-class.html) and the [official package:http retry guide](https://dart.googlesource.com/http.git/+/4afc7f40799368391d5f7e4720c6ea0bd8c0efbb/pkgs/http/README.md).
