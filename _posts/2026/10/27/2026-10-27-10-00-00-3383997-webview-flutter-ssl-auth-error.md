---
layout: post
title: "webview_flutter SSL Auth Error - Stop Bypassing Certificate Failures in Flutter"
description: "Handle webview_flutter 4.13+ recoverable SSL certificate errors with a fail-closed callback, useful logging, and a safe test-only exception policy."
date: 2026-10-27
tags: [webview_flutter, networking, security, Android, iOS]
comments: true
share: true
---

![Flutter webview_flutter SSL certificate error handling](https://images.unsplash.com/photo-1563013544-824ae1b704d3?auto=format&fit=crop&w=1600&q=80)

If a Flutter app embeds a payment, account, or internal web page, the safer production choice is to cancel an unexpected `webview_flutter` SSL error and show a recoverable failure state. Calling `proceed()` is appropriate only for a controlled test environment with a deliberately invalid certificate. This applies to `webview_flutter` 4.13.0 and later, where recoverable SSL callbacks are exposed; it does not replace fixing an expired certificate or a broken server chain.

## Why the callback changes the failure boundary

Before this API, teams often handled a blank WebView through `onWebResourceError`, or copied native examples that accepted every certificate. Those approaches lose the distinction between a page that cannot be trusted and an ordinary network timeout.

| Situation | Safe action | Why |
|---|---|---|
| Production host has an invalid or expired certificate | `cancel()` | Prevents untrusted content from reaching the WebView |
| Local test server uses a self-signed certificate | `proceed()` only behind a debug flag | Keeps the exception explicit and non-production |
| Certificate is valid but the page cannot connect | Handle `onWebResourceError` | This is a transport or server failure, not a trust override |

The package API requires the host app to call either `cancel()` or `proceed()`. Leaving the callback undecided is not a useful policy. The certificate object can help diagnostics, but a matching hostname alone is not enough reason to trust it.

## A fail-closed NavigationDelegate

Keep the decision in one place. The example records only non-secret diagnostics and never logs cookies, authorization headers, or certificate contents.

```dart
import 'package:flutter/foundation.dart';
import 'package:webview_flutter/webview_flutter.dart';

NavigationDelegate buildNavigationDelegate({required bool allowTestCertificate}) {
  return NavigationDelegate(
    onSslAuthError: (SslAuthError error) async {
      debugPrint(
        'WebView SSL failure: certificate=${error.certificate != null}',
      );

      if (kDebugMode && allowTestCertificate) {
        await error.proceed();
        return;
      }

      await error.cancel();
    },
    onWebResourceError: (WebResourceError error) {
      debugPrint(
        'WebView resource failure: type=${error.errorType} '
        'description=${error.description}',
      );
    },
  );
}
```

The exact callback name is `onSslAuthError` on `NavigationDelegate`; the lower-level Android and WebKit platform delegates expose the historical spelling `setOnSSlAuthError`. Mixing those examples is a common compile error during a 4.x migration.

## Test the policy instead of only opening the page

Use a test-only host with an intentionally invalid certificate, then verify that the WebView does not display authenticated content. A production-like check should cover the following paths:

- valid certificate and valid chain: page loads normally;
- expired, hostname-mismatched, or self-signed certificate: callback fires and `cancel()` runs;
- debug-only exception flag: `proceed()` is reachable only in a debug build;
- ordinary DNS, timeout, and HTTP 500 failures: they reach the normal resource or HTTP error handlers;
- redirect to an unexpected host: the navigation policy rejects it before any certificate exception is considered trusted.

Do not use `proceed()` as a workaround for a staging server that is reachable from production users. Fix the staging certificate, or make the staging build point to a separate host and keep the exception behind a compile-time or environment guard that cannot be enabled by a release configuration.

## Short checklist

| Check | Decision |
|---|---|
| Package version | `webview_flutter` 4.13.0+ |
| Default on SSL error | Cancel |
| Test exception | Debug-only and opt-in |
| Logs | Host and error category only |
| Release verification | Invalid certificate must not render the page |

`onSslAuthError` is a narrow recovery hook, not a certificate-pinning system. Treat it as an explicit trust boundary: cancel by default, make test exceptions impossible in release builds, and keep server certificate repair separate from WebView navigation code.

Sources checked on 2026-10-27: [webview_flutter package documentation](https://pub.dev/packages/webview_flutter), [webview_flutter changelog](https://pub.dev/packages/webview_flutter/changelog), [SslAuthError API](https://pub.dev/documentation/webview_flutter/latest/webview_flutter/SslAuthError-class.html), and [AndroidSslAuthError API](https://pub.dev/documentation/webview_flutter_android/latest/webview_flutter_android/AndroidSslAuthError-class.html).
