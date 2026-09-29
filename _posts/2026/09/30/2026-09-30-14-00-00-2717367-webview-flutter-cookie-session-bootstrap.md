---
layout: post
title: "webview_flutter Cookies Before loadRequest - Fix Flutter WebView Session Bootstrap"
description: "Fix webview_flutter login sessions that appear logged out by setting WebViewCookie before loadRequest, matching the host and path, and keeping async initialization outside the widget build."
date: 2026-09-30
tags: [webview_flutter, networking, authentication, Android, iOS]
comments: true
share: true
---

![webview_flutter cookie session bootstrap before loadRequest](https://pub.dev/static/img/pub-dev-icon.svg)

If a Flutter `webview_flutter` page opens as logged out even though the native app has a session token, set the WebView cookie and await that operation before calling `loadRequest`. This pattern fits apps that already receive a server session cookie or short-lived handoff token. It does not make an arbitrary native OAuth token safe to expose to a web page, and it does not replace a proper browser-based OAuth flow.

## The failure is usually an ordering problem

`WebViewController.loadRequest()` can start navigation immediately. If cookie setup happens after it, the first document request has already been sent without the session. A later cookie write may be correct, but the page remains on its logged-out branch until it is loaded again.

There are three values to check together:

| Check | Typical mistake | Result |
| --- | --- | --- |
| Host | Cookie is set for `api.example.com`, page is `app.example.com` | Cookie is not sent |
| Path | Cookie uses `/api`, page starts at `/dashboard` | Cookie is out of scope |
| Scheme | `Secure` cookie is tested over `http://` | Cookie is rejected |

The official `webview_flutter` example follows the important sequence: call `WebViewCookieManager.setCookie`, then call `loadRequest`. Keep that sequence in one initialization method so a widget rebuild cannot accidentally create a second, unauthenticated navigation.

## Keep cookie setup and navigation in one async boundary

The controller below is created once. The first request does not begin until the cookie write has completed.

```dart
import 'package:flutter/material.dart';
import 'package:webview_flutter/webview_flutter.dart';

class AccountWebView extends StatefulWidget {
  const AccountWebView({super.key, required this.sessionId});

  final String sessionId;

  @override
  State<AccountWebView> createState() => _AccountWebViewState();
}

class _AccountWebViewState extends State<AccountWebView> {
  late final Future<WebViewController> _controller = _createController();

  Future<WebViewController> _createController() async {
    final cookieManager = WebViewCookieManager();
    final controller = WebViewController()
      ..setJavaScriptMode(JavaScriptMode.unrestricted);

    await cookieManager.setCookie(
      WebViewCookie(
        name: 'session_id',
        value: widget.sessionId,
        domain: 'app.example.com',
        path: '/',
      ),
    );

    await controller.loadRequest(
      Uri.parse('https://app.example.com/dashboard'),
    );
    return controller;
  }

  @override
  Widget build(BuildContext context) {
    return FutureBuilder<WebViewController>(
      future: _controller,
      builder: (context, snapshot) {
        if (!snapshot.hasData) {
          return const Center(child: CircularProgressIndicator());
        }
        return WebViewWidget(controller: snapshot.data!);
      },
    );
  }
}
```

The `FutureBuilder` is only displaying the one controller future; it is not performing the setup in `build`. If the session can change, give the widget a new `Key` or expose an explicit reload method that updates the cookie and then reloads the same URL. Do not silently set a new cookie on every rebuild.

## What to verify before blaming the package

Use this checklist with a test account and a non-sensitive cookie value.

- Confirm that the cookie domain matches the URL actually loaded. A subdomain mismatch is enough to produce a clean, logged-out page.
- Use `https://` when the server requires a secure cookie. The Android implementation documents that a secure cookie needs an HTTPS URL.
- Set `path: '/'` unless the server intentionally restricts the session to a narrower path.
- Set the cookie before the first `loadRequest`; do not rely on `onPageFinished` for bootstrap.
- Check the server’s `Set-Cookie` policy. `HttpOnly` and `Secure` are server-side protections; a native token copied into a page-visible cookie changes the threat model.
- Treat `sessionId` as sensitive. Avoid logging the value, putting it in analytics events, or embedding it in a URL.

If the web server itself returns `Set-Cookie` after a login page, let the WebView’s cookie store receive that response. The pre-seeding example is for a deliberate handoff from a native authentication layer, not a way to bypass the server’s session rules.

## Short decision guide

| Situation | Choice |
| --- | --- |
| Native app already owns a server-approved session cookie | Set it, await it, then load the WebView URL |
| Web login should establish the session | Load the login URL and let the response set cookies |
| Native access token must call an API | Keep it in native networking code; do not inject it into page JavaScript |
| Cookie works on one screen but not another | Compare host, path, scheme, and cookie expiry |

The practical fix is small: one `WebViewCookieManager`, one awaited cookie write, and one first navigation after it. The host and security attributes decide whether that small sequence actually authenticates the page.

Sources: [webview_flutter package documentation](https://pub.dev/packages/webview_flutter), [official webview_flutter example](https://pub.dev/packages/webview_flutter/versions/4.14.0/example), and [Android cookie API documentation](https://chromium.googlesource.com/external/github.com/flutter/packages/%2B/43a42d1ada651b0d8f430c5884897785dc4f43f2/packages/webview_flutter/webview_flutter_android/lib/src/android_webview.dart) (checked 2026-09-30).
