---
layout: post
title: "Dio QueuedInterceptor - Fix Flutter Token Refresh Deadlocks Under Concurrent 401s"
description: "Handle concurrent Dio 401 responses in Flutter without duplicate token refreshes or QueuedInterceptor deadlocks, using a shared refresh future and a separate client."
date: 2026-09-18
tags: [dio, networking, authentication, concurrency, Flutter, testing]
comments: true
share: true
---

![Dio token refresh flow for concurrent Flutter requests](/assets/images/go-router-refresh-redirect-auth.png)

The key boundary is separating an expired authenticated request from the refresh request that obtains its replacement token.

If a Flutter app sends several Dio requests at once, use a shared refresh future and a separate Dio client for the refresh call. This is safer than putting the whole retry flow inside `QueuedInterceptor`: the queue can serialize 401 handling, but a retry made through the same queued client can wait for the interceptor that is waiting for the retry. A mobile app with only public endpoints does not need this pattern; it matters when an expired access token can invalidate several requests together.

## The failure has two different causes

Suppose `Future.wait` starts three authenticated requests with the same expired token.

| Design | What happens under three 401 responses | Risk |
| --- | --- | --- |
| Plain `Interceptor`, refresh in every `onError` | Three refresh calls race | Refresh-token rotation or rate limits can log the user out |
| `QueuedInterceptor`, retry through the same Dio | Refresh is serialized, but the retry re-enters the waiting queue | Deadlock or a queue that appears stuck |
| Plain `Interceptor`, shared refresh future, separate refresh Dio | One refresh is shared; each failed request retries once | Predictable failure and logout behavior |

`QueuedInterceptor` is useful when request interceptors must run sequentially. It is not a general-purpose “refresh only once” lock. The token refresh operation has a different requirement: requests that observe the same old token should await one operation, then compare the token again before starting another operation.

## Keep the refresh client outside the authenticated pipeline

The following example is deliberately small. `TokenStore` represents secure persistence in the application; it is not an in-memory substitute for a real credential store.

```dart
import 'package:dio/dio.dart';

class TokenPair {
  const TokenPair(this.accessToken, this.refreshToken);

  final String accessToken;
  final String refreshToken;
}

abstract interface class TokenStore {
  String? get accessToken;
  String? get refreshToken;
  Future<void> save(TokenPair pair);
  Future<void> clear();
}

class AuthInterceptor extends Interceptor {
  AuthInterceptor({required this.api, required this.refreshDio, required this.store});

  final Dio api;
  final Dio refreshDio;
  final TokenStore store;
  Future<TokenPair>? _refreshing;

  @override
  void onRequest(RequestOptions options, RequestInterceptorHandler handler) {
    final token = store.accessToken;
    if (token != null) {
      options.headers['Authorization'] = 'Bearer $token';
    }
    handler.next(options);
  }

  @override
  Future<void> onError(
    DioException error,
    ErrorInterceptorHandler handler,
  ) async {
    final request = error.requestOptions;
    final oldToken = request.headers['Authorization'] as String?;
    final canRetry = error.response?.statusCode == 401 &&
        request.extra['authRetried'] != true &&
        oldToken != null &&
        store.refreshToken != null;

    if (!canRetry) {
      handler.next(error);
      return;
    }

    try {
      final currentToken = 'Bearer ${store.accessToken}';
      if (oldToken != currentToken && store.accessToken != null) {
        // Another request already refreshed the token.
      } else {
        _refreshing ??= _refreshAccessToken().whenComplete(() {
          _refreshing = null;
        });
        await _refreshing;
      }

      final retry = request.copyWith(
        headers: {
          ...request.headers,
          'Authorization': 'Bearer ${store.accessToken}',
        },
        extra: {...request.extra, 'authRetried': true},
      );
      final response = await api.fetch<dynamic>(retry);
      handler.resolve(response);
    } on DioException catch (refreshError) {
      if (refreshError.response?.statusCode == 401) {
        await store.clear();
      }
      handler.next(refreshError);
    } catch (_) {
      handler.next(error);
    }
  }

  Future<TokenPair> _refreshAccessToken() async {
    final refreshToken = store.refreshToken!;
    final response = await refreshDio.post<Map<String, dynamic>>(
      '/auth/refresh',
      data: {'refresh_token': refreshToken},
    );
    final data = response.data!;
    final pair = TokenPair(
      data['access_token'] as String,
      data['refresh_token'] as String? ?? refreshToken,
    );
    await store.save(pair);
    return pair;
  }
}
```

The important boundaries are easy to miss:

- `refreshDio` has no `AuthInterceptor`, so a failed refresh cannot recursively refresh itself.
- `_refreshing` is a shared `Future`, not a boolean. Every waiting request receives the same result or the same failure.
- `oldToken` is compared with the token currently in the store. If request A refreshes while request B is handling its old 401, B reuses A's token instead of refreshing again.
- `authRetried` limits each original request to one replay. Without it, a server-side revocation can create an infinite 401 loop.

Do not use `request.headers['Authorization']` as the only token source for a retry. It contains the stale bearer value by definition. Also keep the refresh endpoint out of global business-error handling if that handler displays duplicate session-expired dialogs.

## Test the race, not only the happy path

The smallest useful test controls the refresh response and starts multiple requests before completing it.

```dart
test('three expired requests share one refresh', () async {
  final refreshCalls = <void>[];
  // Configure a fake adapter: three first calls return 401, then retries 200.
  // Complete the refresh response only after all three requests are pending.

  await Future.wait([
    api.get('/profile'),
    api.get('/inbox'),
    api.get('/settings'),
  ]);

  expect(refreshCalls, hasLength(1));
});
```

Add two more cases: a refresh 401 must clear credentials and reject all waiting calls, and a retried request receiving 401 must not call refresh again. If the app supports cancellation, decide whether a cancelled original request should still be replayed; a shared refresh can finish for the other requests while that one request is discarded.

## Practical decision checklist

Use this pattern when the app has refresh-token rotation, concurrent startup calls, or a shared API client used by several repositories. A plain interceptor is enough when the backend never expires access tokens during a session. Choose `QueuedInterceptor` for genuinely sequential request mutation, such as obtaining a CSRF token once, but keep token refresh and retry re-entry outside that queue unless you can prove the control flow cannot wait on itself.

The reproducible rule is: one old token → one shared refresh future → one retry per request → a different client for the refresh call. That sequence removes both duplicate refresh storms and the most common Dio interceptor deadlock.

References: [Dio API documentation](https://pub.dev/documentation/dio/latest/), [Dio `QueuedInterceptor` documentation](https://pub.dev/documentation/dio/latest/dio/QueuedInterceptor-class.html), [Dio discussion on concurrent token updates](https://github.com/cfug/dio/discussions/2302), and [Dio issue on queued retry deadlocks](https://github.com/cfug/dio/issues/1612).
