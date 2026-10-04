---
layout: post
title: "Dio Multipart Retry - Rebuild Flutter File Uploads After Token Refresh"
description: "Fix Dio multipart uploads that fail after a 401 token refresh by cloning FormData, isolating the refresh request, and limiting retries to safe cases."
date: 2026-10-05
tags: [dio, networking, authentication, migration, testing]
comments: true
share: true
---
![Dio multipart upload retry after token refresh](../../../assets/images/dio-multipart-token-refresh-retry.png)

The lower file packet is the rebuilt upload that succeeds after the expired-token path is refreshed.

The important choice is simple: if a Flutter `dio` upload gets a `401`, refresh the token with a separate client and resend a cloned `FormData`; do not send the same multipart object again. This applies to apps that upload a file while an access token can expire. It does not mean every failed `POST` should be retried, and it is not a substitute for an idempotency key when the server may have already accepted the upload.

## Why the second upload disappears

`FormData` is turned into a stream when Dio sends it. After that stream is finalized, reusing the original object can produce `Cannot finalize` or an empty second request. The failure is easy to miss because a JSON request often retries successfully while a file upload does not.

The Dio API documents `MultipartFile.clone()` and `FormData.clone()` specifically for resending a failed request. Its package documentation also recommends creating a new `FormData` or `MultipartFile` for repeated requests. The relevant distinction is the request body, not just the `Authorization` header.

| Situation | Reuse original body? | Safer action |
| --- | ---: | --- |
| GET returns 401 | No body | Refresh once, then repeat the request |
| JSON POST returns 401 | Usually no stream | Recreate the body and use an idempotency key when needed |
| Multipart upload returns 401 before acceptance | No | Clone `FormData`, update the token, retry once |
| Timeout after an unknown server result | No automatic retry | Query upload status or use a server-side idempotency key |

## Keep token refresh out of the retry loop

Use a second Dio instance for the refresh endpoint. If the refresh call uses the same interceptor, an expired refresh token can call the interceptor again and create a loop. The upload client should also mark the request after one authentication retry.

The following is a reusable shape. `auth.refreshAccessToken()` represents the refresh client and returns the new access token; it is not shown as a global mutable singleton so the retry policy stays visible.

```dart
import 'package:dio/dio.dart';

class AuthRetryInterceptor extends QueuedInterceptor {
  AuthRetryInterceptor({
    required this.api,
    required this.auth,
  });

  final Dio api;
  final AuthSession auth;

  @override
  Future<void> onError(
    DioException error,
    ErrorInterceptorHandler handler,
  ) async {
    final response = error.response;
    final request = error.requestOptions;
    final alreadyRetried = request.extra['authRetry'] == true;

    if (response?.statusCode != 401 || alreadyRetried) {
      handler.next(error);
      return;
    }

    // A refresh request must use a client without this interceptor.
    final accessToken = await auth.refreshAccessToken();
    if (accessToken == null) {
      handler.next(error);
      return;
    }

    final retryData = request.data is FormData
        ? (request.data as FormData).clone()
        : request.data;

    final retryOptions = request.copyWith(
      data: retryData,
      extra: {...request.extra, 'authRetry': true},
      headers: {
        ...request.headers,
        'Authorization': 'Bearer $accessToken',
      },
    );

    try {
      final retried = await api.fetch<dynamic>(retryOptions);
      handler.resolve(retried);
    } on DioException catch (retryError) {
      handler.next(retryError);
    }
  }
}

abstract interface class AuthSession {
  Future<String?> refreshAccessToken();
}
```

`QueuedInterceptor` matters when several requests receive `401` together. It serializes interceptor access, but it is not a complete refresh-token lock: the session layer still needs to coalesce simultaneous refresh calls if the backend invalidates the old refresh token after the first use.

## Build the multipart body at the request boundary

For uploads created from a picker, keep the source bytes or a stable file path and make the `FormData` inside the operation. That makes a later retry deterministic and avoids retaining a finalized object in a repository field.

```dart
Future<Response<dynamic>> uploadDocument({
  required Dio api,
  required List<int> bytes,
  required String filename,
}) {
  final body = FormData.fromMap({
    'document': MultipartFile.fromBytes(bytes, filename: filename),
  });

  return api.post('/documents', data: body);
}
```

If the file is large, keeping all bytes in memory may be the wrong trade-off. Store a temporary path and recreate `MultipartFile.fromFile` for each new logical attempt instead. Do not clone a stream whose source has already been closed.

## Retry checklist

- Refresh with a client that has no auth-retry interceptor.
- Retry only once per request; tag it in `RequestOptions.extra`.
- Clone `FormData` and `MultipartFile`, or rebuild them from stable bytes/path data.
- Treat a timeout after upload as an unknown server result, not as a safe retry.
- Add an idempotency key when the API can deduplicate repeated `POST` operations.
- Test two concurrent `401` responses and a refresh failure separately.

The short rule is: a new token does not make an old multipart stream reusable. Refresh the credential, recreate the body, and make the server-side outcome safe to repeat. The [Dio `MultipartFile` API](https://pub.dev/documentation/dio/latest/dio/MultipartFile-class.html) documents `clone()` for this retry case, while the [FormData API](https://pub.dev/documentation/dio/latest/dio/FormData-class.html) documents cloning a finalized form for resending.
