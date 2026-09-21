---
layout: post
title: "Dio Concurrent Request Deduplication - Prevent Flutter Completer Errors"
description: "Fix a Dio deduplication interceptor that breaks under concurrent requests by sharing futures safely, converting errors for late subscribers, and cleaning pending entries deterministically."
date: 2026-09-22
tags: [dio, networking, concurrency, testing, Flutter]
comments: true
share: true
---
![Dio concurrent request deduplication flow](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

If several Flutter widgets request the same Dio resource at startup, a request-deduplication interceptor should share one in-flight future. The production trap is completing that future with an error before a later duplicate request subscribes to it. Dart can then report an unhandled asynchronous error, dispatch the duplicate separately, or leave it waiting forever. Apps without duplicate requests do not need this interceptor; a normal Dio client is simpler.

## The failure is a timing problem, not just a map lookup

The usual implementation stores a `Completer` by request key. The first request goes to the adapter. A second request finds the same key and awaits the completer. That looks safe until the adapter fails immediately. `onError` completes the completer before the second request has attached its error handler. The result depends on event-loop timing.

The Dio issue [#2565](https://github.com/cfug/dio/issues/2565) describes this exact shape with Dio 5.10.0 on iOS: synchronous identical requests plus an immediately failing adapter. The issue was marked fixed, but custom interceptors still need the same lifecycle rules because their `Completer` and key cleanup are application code.

| Decision | Safe rule | Why |
| --- | --- | --- |
| Request key | Include method, URL, query, and relevant body | Two requests with different authorization or payload must not share a response |
| Shared result | Store a `Future`, not an exposed mutable `Completer` | Subscribers cannot complete or replace the owner’s state |
| Failure | Attach an error path before returning the shared future | Late subscribers receive a Dio error through their own handler |
| Cleanup | Remove only the same future that was inserted | A late completion must not delete a newer request |

## Keep ownership inside the interceptor

The following version keeps a private future per key. The first request owns the adapter call; duplicates subscribe to the same result. `whenComplete` removes the entry only when it still points to that future.

```dart
import 'package:dio/dio.dart';

class RequestDeduplicationInterceptor extends Interceptor {
  RequestDeduplicationInterceptor(this.dio);

  final Dio dio;
  final _pending = <String, Future<Response<dynamic>>>{};

  String _key(RequestOptions options) {
    final query = options.queryParameters.entries.toList()
      ..sort((a, b) => a.key.compareTo(b.key));
    return '${options.method}:${options.uri}|$query|${options.data}';
  }

  @override
  void onRequest(RequestOptions options, RequestInterceptorHandler handler) {
    final key = _key(options);
    final existing = _pending[key];
    if (existing != null) {
      existing.then(
        handler.resolve,
        onError: (Object error, StackTrace stackTrace) {
          handler.reject(
            error is DioException
                ? error.copyWith(requestOptions: options)
                : DioException(
                    requestOptions: options,
                    error: error,
                    stackTrace: stackTrace,
                  ),
          );
        },
      );
      return;
    }

    // The adapter is invoked after the interceptor chain continues.
    late final Future<Response<dynamic>> request;
    request = dio.fetch<dynamic>(options).whenComplete(() {
      if (identical(_pending[key], request)) {
        _pending.remove(key);
      }
    });
    _pending[key] = request;
    request.then(handler.resolve, onError: (Object error, StackTrace stackTrace) {
      handler.reject(
        error is DioException
            ? error
            : DioException(
                requestOptions: options,
                error: error,
                stackTrace: stackTrace,
              ),
      );
    });
  }
}
```

The interceptor receives the Dio client through its constructor. Do not attach it to a second internal client. If your Dio version or adapter setup makes `dio.fetch` re-enter the same interceptor, use a private adapter/client boundary or pass an `Options` flag that the interceptor checks before deduplicating. The boundary matters more than the exact field name.

There are three details worth preserving. First, a duplicate gets a new `DioException` request context, so its path and headers remain diagnosable. Second, the cleanup callback compares futures by identity. Without that guard, request A can finish late and remove request B, which already reused the same key. Third, the key must not include unstable values such as a timestamp; that silently disables deduplication.

## Do not deduplicate every request

Use the pattern for idempotent reads such as `GET /profile` or a configuration document. Do not share `POST`, payment, upload, or mutation requests unless the server contract explicitly defines an idempotency key. Also decide whether auth headers belong in the key. If the same URL can return different data for different users, the user or credential scope must be part of the key, or the interceptor must be scoped per authenticated client.

### Reproduction checklist

1. Start two identical requests in the same synchronous turn with `Future.wait`.
2. Replace the adapter with one that fails before the second request subscribes.
3. Assert that both callers complete with an error and neither remains pending.
4. Start a second request after the first failure and assert that it reaches the adapter again.
5. Repeat with different query parameters, users, and request bodies.

The useful invariant is simple: one key has at most one in-flight read, every subscriber receives either the same response or its own contextual error, and the key is available for a later retry after completion. That is the part to test; a happy-path test with a successful adapter will not expose the race.

Sources: [Dio API documentation](https://pub.dev/documentation/dio/latest/), [Dio issue #2565](https://github.com/cfug/dio/issues/2565).
