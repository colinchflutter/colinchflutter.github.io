---
layout: post
title: "webview_flutter NavigationDelegate - Keep iOS Embeds Inside and Open Only Real External Links"
description: "Fix webview_flutter navigationDelegate callbacks that mistake iframe resources for user clicks. Filter main-frame requests, validate URL schemes, and launch external links safely with url_launcher."
date: 2026-10-15
tags: [webview_flutter, url_launcher, networking, iOS, Android]
comments: true
share: true
---

![webview_flutter navigation delegate filtering main-frame requests](https://pub.dev/static/img/pub-dev-icon.svg)

If a Flutter WebView should keep article pages and embeds inside the app but open clicked external links in the system browser, inspect `NavigationRequest.isMainFrame` before calling `url_launcher`. This fits content apps, help centers, and hybrid checkout pages that contain iframes or JavaScript-loaded resources. It does not fit an app that must sandbox every third-party origin; that app needs a stricter origin policy and probably a dedicated WebView plugin.

The common bug is an overly broad callback: every URL that is not the base URL is sent to an external browser. On iOS, subframes and side-loaded content can reach the navigation delegate, so YouTube embeds, consent tools, ads, or a login provider may be treated like a user click. The result is either a browser opening during page load or an embed that never renders.

## The decision must use two separate questions

`webview_flutter` exposes the pending URL and whether it belongs to the main frame. The URL answers “where is this request going?” The frame flag answers “is this the page-level navigation that can replace the current document?” They are not interchangeable.

| Request | Keep in WebView? | External launch? | Reason |
|---|---:|---:|---|
| Main frame on the app’s trusted origin | Yes | No | Normal in-app navigation |
| Subframe on a trusted embed origin | Yes | No | Required for iframe or player content |
| Main frame on an untrusted `https` origin | No | Yes, if policy allows | A real page-level external link |
| `http`, `javascript:`, `file:`, or malformed URL | No | No | Do not hand unsafe or unsupported schemes to the OS |

The old property name `isForMainFrame` appears in older issue reports and examples. Current `webview_flutter` 4.14.1 documents the property as `isMainFrame`; copy the API that matches the version resolved by your project instead of mixing snippets from both generations.

## A production-shaped delegate

Keep the policy in one class or function. That makes the allowlist reviewable and prevents a widget rebuild from creating slightly different navigation rules.

```dart
import 'package:url_launcher/url_launcher.dart';
import 'package:webview_flutter/webview_flutter.dart';

class WebViewNavigationPolicy {
  WebViewNavigationPolicy({
    required this.appHost,
    this.embedHosts = const <String>{
      'www.youtube.com',
      'www.youtube-nocookie.com',
      'player.vimeo.com',
    },
  });

  final String appHost;
  final Set<String> embedHosts;

  Future<NavigationDecision> decide(NavigationRequest request) async {
    final uri = Uri.tryParse(request.url);
    if (uri == null || uri.host.isEmpty) {
      return NavigationDecision.prevent;
    }

    final isWeb = uri.scheme == 'https' || uri.scheme == 'http';
    if (!isWeb) {
      return NavigationDecision.prevent;
    }

    final isTrustedPage = uri.host == appHost;
    final isTrustedEmbed = embedHosts.contains(uri.host);
    if (isTrustedPage || isTrustedEmbed) {
      return NavigationDecision.navigate;
    }

    // A subframe is a resource inside the page, not automatically a click.
    if (!request.isMainFrame) {
      return NavigationDecision.prevent;
    }

    await launchUrl(
      uri,
      mode: LaunchMode.externalApplication,
    );
    return NavigationDecision.prevent;
  }
}
```

The final conditional deliberately returns `prevent` in both branches. The external launch is a side effect; allowing the WebView to continue after launching would create two navigations. In a real app, log the `false` result or show a small error state rather than allowing the URL as a fallback.

Attach one policy to the controller:

```dart
late final WebViewController controller;

void configureWebView() {
  final policy = WebViewNavigationPolicy(appHost: 'docs.example.com');

  controller = WebViewController()
    ..setJavaScriptMode(JavaScriptMode.unrestricted)
    ..setNavigationDelegate(
      NavigationDelegate(
        onNavigationRequest: policy.decide,
      ),
    )
    ..loadRequest(Uri.parse('https://docs.example.com/start'));
}
```

Do not use `startsWith('https://docs.example.com')` as the host check. A value such as `https://docs.example.com.attacker.test` passes that string test but is not your origin. Parse the URI and compare `scheme` and `host`; add a port check too if your service uses a non-default port.

## Why a giant allowlist is usually the wrong repair

An allowlist of every iframe provider grows with the content team’s publishing habits. A better boundary is:

1. Allow the app’s own origin.
2. Allow only the embed origins the product intentionally supports.
3. Block subframes from unknown origins unless the product explicitly needs them.
4. Send only main-frame `http`/`https` navigations to the external browser.

This is a product decision, not a universal security rule. If your page legitimately loads a consent manager, payment provider, or identity provider inside a frame, add that origin deliberately and document why. Avoid `*.example.com` matching unless you also control every subdomain; a delegated or user-content subdomain can turn a broad rule into a trust boundary failure.

`url_launcher` also has platform constraints. `launchUrl` accepts a `Uri`, and `LaunchMode.externalApplication` asks the operating system to hand the URL to another application. If the app supports `canLaunchUrl` checks, Android 11+ and iOS may require query declarations for the schemes being queried. A failed launch should remain a blocked navigation, not a silent navigation into an unknown WebView page.

## Reproduction checklist

Use a page containing all four cases before shipping the delegate:

```text
[ ] Same-origin link stays in the WebView.
[ ] YouTube/Vimeo iframe does not open Safari or Chrome during page load.
[ ] A clicked external HTTPS link opens once in the system browser.
[ ] javascript:, file:, malformed, and unknown-scheme URLs are blocked.
[ ] A failed launch does not navigate the WebView to the external URL.
[ ] Android and iOS logs include isMainFrame and the parsed host.
```

The Flutter issue tracker contains a reproducible iOS report where side-loaded content and embeds entered the navigation callback, while a main-frame check reduced the false external launches. The report is old, but the failure mode is still relevant: the mistake is in treating a navigation callback as proof of a user click.

The practical fix is therefore small but specific: parse the URL, decide trust by origin, treat `isMainFrame` as a page-level boundary, and launch only approved external navigations. If the app needs a strict per-origin policy, keep the policy data separate from the widget so a content or security review can change it without rewriting WebView lifecycle code.

Sources checked 2026-10-15: [webview_flutter API documentation](https://pub.dev/documentation/webview_flutter/latest/webview_flutter/), [NavigationRequest.isMainFrame](https://pub.dev/documentation/webview_flutter/latest/webview_flutter/NavigationRequest-class.html), [Flutter issue #73242](https://github.com/flutter/flutter/issues/73242), and [url_launcher documentation](https://pub.dev/packages/url_launcher).
