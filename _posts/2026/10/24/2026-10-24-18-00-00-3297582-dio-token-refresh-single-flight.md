---
layout: post
title: "Dio Token Refresh - Prevent Flutter 401 Retry Loops and Duplicate Refresh Calls"
description: "A production-safe Dio interceptor pattern for Flutter that shares one token refresh across concurrent 401 responses, retries each request once, and avoids replaying unsafe uploads."
date: 2026-10-24
tags: [dio, networking, authentication, concurrency, Flutter]
comments: true
share: true
---

![Dio token refresh flow for concurrent Flutter requests](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=800&q=80)

If a Flutter screen sends several Dio requests at startup, the safest token-refresh policy is **one shared refresh Future, one replay per request, and no retry of the refresh endpoint itself**. This fits authenticated `GET` and idempotent reads. It does not automatically make a payment, file upload, or other one-shot `POST` safe to replay.

## Why the usual interceptor fails

When five requests receive `401` together, a naive `onError` starts five refresh calls. A slightly improved version shares the refresh call but retries forever when the refresh request also passes through the same interceptor. A third version retries the original request without marking it, so the new `401` starts another refresh.

Dio documents that ordinary interceptors can run concurrently; `QueuedInterceptor` is available when sequential access is actually required. The fix below uses a shared Future for the refresh operation and an `extra` flag for the retry budget. The refresh client is separate, so its failure cannot recursively enter the authenticated interceptor.

| Situation | Action | Reason |
| --- | --- | --- |
| First authenticated `401` | Start `_refreshing` | Only one refresh request is needed |
| Another `401` during refresh | Await the same Future | Prevents a refresh storm |
| Refresh succeeds | Save the new access token, replay once | Original callers receive the result |
| Refresh fails permanently | Reject every waiting request | Do not hide an expired session |
| Upload uses a one-shot stream | Do not replay automatically | The body may already be consumed |

## A single-flight interceptor

The refresh service returns a new access token or throws. It must not use the `api` client that owns this interceptor.

```dart
import 'package:dio/dio.dart';

class AuthInterceptor extends Interceptor {
  AuthInterceptor({
    required this.api,
    required this.refreshDio,
    required this.readAccessToken,
    required this.refreshAccessToken,
    required this.saveAccessToken,
    required this.clearSession,
  });

  final Dio api;
  final Dio refreshDio;
  final Future<String?> Function() readAccessToken;
  final Future<String> Function(Dio refreshDio) refreshAccessToken;
  final Future<void> Function(String token) saveAccessToken;
  final Future<void> Function() clearSession;

  Future<String>? _refreshing;

  @override
  Future<void> onRequest(
    RequestOptions options,
    RequestInterceptorHandler handler,
  ) async {
    final token = await readAccessToken();
    if (token != null && token.isNotEmpty) {
      options.headers['Authorization'] = 'Bearer $token';
    }
    handler.next(options);
  }

  @override
  Future<void> onError(
    DioException error,
    ErrorInterceptorHandler handler,
  ) async {
    final options = error.requestOptions;
    final is401 = error.response?.statusCode == 401;
    final isRefreshCall = options.extra['authRefreshCall'] == true;
    final alreadyRetried = options.extra['authRetried'] == true;

    if (!is401 || isRefreshCall || alreadyRetried) {
      handler.next(error);
      return;
    }

    try {
      final refreshFuture = _refreshing ??= _refresh();
      final token = await refreshFuture;
      if (identical(_refreshing, refreshFuture)) {
        _refreshing = null;
      }

      final replayOptions = options.copyWith(
        headers: {
          ...options.headers,
          'Authorization': 'Bearer $token',
        },
        extra: {
          ...options.extra,
          'authRetried': true,
        },
      );
      final response = await api.fetch<dynamic>(replayOptions);
      handler.resolve(response);
    } catch (refreshError, stackTrace) {
      if (identical(_refreshing, refreshFuture)) {
        _refreshing = null;
      }
      await clearSession();
      handler.reject(
        DioException(
          requestOptions: options,
          error: refreshError,
          stackTrace: stackTrace,
        ),
      );
    }
  }

  Future<String> _refresh() async {
    final token = await refreshAccessToken(refreshDio);
    await saveAccessToken(token);
    return token;
  }
}
```

The `identical` check matters when refresh A finishes after refresh B has already been assigned. Without it, an old completion can clear a newer in-flight Future. The `authRetried` flag is also a request-local circuit breaker: a bad new token produces one final `401`, not an infinite loop.

Create the clients so the refresh endpoint cannot trigger the authenticated interceptor:

```dart
final refreshDio = Dio(BaseOptions(baseUrl: apiBaseUrl));
final api = Dio(BaseOptions(baseUrl: apiBaseUrl));

api.interceptors.add(AuthInterceptor(
  api: api,
  refreshDio: refreshDio,
  readAccessToken: tokenStore.readAccessToken,
  refreshAccessToken: (client) async {
    final refreshToken = await tokenStore.readRefreshToken();
    final response = await client.post('/auth/refresh', data: {
      'refresh_token': refreshToken,
    }, options: Options(extra: {'authRefreshCall': true}));
    return response.data['access_token'] as String;
  },
  saveAccessToken: tokenStore.saveAccessToken,
  clearSession: tokenStore.clearSession,
));
```

## What not to replay

`api.fetch(options)` reuses the original request data. JSON bodies and repeatable byte buffers are usually straightforward, but a `Stream`, `MultipartFile`, or consumed `FormData` may not be. Dio's own documentation warns that a new `FormData` or `MultipartFile` is needed for repeated requests.

Use one of these policies for non-idempotent operations:

- Mark payment, order creation, and destructive mutations as non-retryable and surface the `401`.
- Buffer a small upload and rebuild `FormData` for the replay.
- Send an idempotency key understood by the server before enabling an automatic replay.

## Release checklist

Before shipping the interceptor, exercise these cases with a fake adapter or mock server:

```text
[ ] Five parallel requests cause exactly one refresh call.
[ ] All five requests receive the refreshed token or the same refresh failure.
[ ] The refresh endpoint is never intercepted as an API request.
[ ] A replayed request can produce only one additional 401.
[ ] A refresh timeout does not erase a still-valid session unless policy says so.
[ ] FormData, streams, payments, and mutations have an explicit replay policy.
[ ] Logs redact access tokens and refresh-token responses.
```

Dio's interceptor API supports resolving a response or rejecting with a `DioException`; use those paths deliberately so the original callers do not hang. If the app has no concurrent authenticated requests, a plain request wrapper is easier to maintain than this interceptor. The extra coordination is justified only when the failure mode is real and reproducible.

Sources: [Dio Interceptor API](https://pub.dev/documentation/dio/latest/dio/Interceptor-class.html), [Dio RequestOptions API](https://pub.dev/documentation/dio/latest/dio/RequestOptions-class.html), [Dio official README](https://github.com/cfug/dio/blob/main/dio/README.md).
