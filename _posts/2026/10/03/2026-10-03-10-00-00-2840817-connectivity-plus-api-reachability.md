---
layout: post
title: "connectivity_plus API Reachability - Stop Treating Wi-Fi as Internet in Flutter"
description: "Use connectivity_plus for transport changes and an API probe for real reachability, with a Flutter pattern that handles resume, captive portals, timeouts, and server failures."
date: 2026-10-03
tags: [connectivity_plus, networking, offline, testing, Android, iOS]
comments: true
share: true
---

![Flutter connectivity_plus network type versus API reachability](/assets/images/connectivity-plus-api-reachability.png)

`connectivity_plus` is useful for knowing whether a device exposes Wi-Fi, mobile, VPN, or another network interface. It is not a reliable answer to “can this app reach its API?”. Use the package to update the UI quickly, but let a request to your own endpoint decide whether syncing can proceed. This split is the right fit for apps with offline banners, retry buttons, or background refresh. It is unnecessary for a screen that simply attempts a request and already handles its timeout and error state well.

## The production bug: two different questions

The common implementation looks like this:

```dart
final result = await Connectivity().checkConnectivity();
final online = result.any((item) => item != ConnectivityResult.none);

if (online) {
  await loadOrders();
} else {
  showOfflineMessage();
}
```

That code answers “is at least one transport reported?”. It does not answer “can the server used by `loadOrders()` be reached?”. A hotel Wi-Fi login page, a broken DNS path, a corporate VPN policy, or a server outage can all make the second answer different.

The package documentation also calls out two lifecycle traps. Android does not deliver connectivity broadcasts to background apps from Android O onward, so the state should be checked again when the app resumes. On iOS, the path monitor can emit transient or unreliable sequences while reconnecting. Treat the stream as a hint for refreshing state, not as proof that a request will succeed.

| Signal | Good for | Do not use it for |
| --- | --- | --- |
| `ConnectivityResult` list | Choosing a network icon, scheduling a re-check, explaining likely transport | Authorizing an API call or declaring all services online |
| Request to your health/API endpoint | Deciding whether this app can reach its dependency | Proving that every third-party service is healthy |
| Actual business request | Final truth for the current operation | Showing a fast global connectivity indicator |

## A small reachability service

Expose a lightweight unauthenticated endpoint such as `/healthz` on the same host and through the same gateway as the API. A successful response, including a normal `401` or `403` when authentication is intentionally required, proves that the network path reached your server. A `500` should be classified as a server problem, not as an offline device.

The following service keeps transport state and application reachability separate. The endpoint is deliberately configurable so tests and staging do not probe a public website.

```dart
import 'dart:async';

import 'package:connectivity_plus/connectivity_plus.dart';
import 'package:flutter/widgets.dart';
import 'package:http/http.dart' as http;

enum NetworkState { noTransport, checking, reachable, serverError }

class NetworkReachability {
  NetworkReachability({
    required Uri healthUri,
    http.Client? client,
    Connectivity? connectivity,
  })  : _healthUri = healthUri,
        _client = client ?? http.Client(),
        _connectivity = connectivity ?? Connectivity();

  final Uri _healthUri;
  final http.Client _client;
  final Connectivity _connectivity;

  Future<NetworkState> check() async {
    final transports = await _connectivity.checkConnectivity();
    final hasTransport = transports.any(
      (item) => item != ConnectivityResult.none,
    );

    if (!hasTransport) return NetworkState.noTransport;

    try {
      final response = await _client
          .get(_healthUri)
          .timeout(const Duration(seconds: 4));

      if (response.statusCode >= 200 && response.statusCode < 500) {
        return NetworkState.reachable;
      }
      return NetworkState.serverError;
    } on TimeoutException {
      return NetworkState.serverError;
    } on Exception {
      return NetworkState.serverError;
    }
  }

  void dispose() => _client.close();
}
```

The `serverError` name is intentional: a timeout does not prove that the device is offline. It could be a slow route, a blocked host, or a gateway failure. If the UI needs more detail, replace the enum with a sealed result containing `transport`, `httpStatus`, and the exception category. Keep the decision boundary explicit instead of converting every failure into `offline`.

## Refresh at the right lifecycle boundary

A stream subscription can update a banner, but it should not trigger an unlimited request loop. Debounce reconnect events, cancel the subscription, and perform a direct check in `didChangeAppLifecycleState` when returning to the foreground.

```dart
class NetworkController with WidgetsBindingObserver {
  NetworkController(this._reachability, {Connectivity? connectivity})
      : _connectivity = connectivity ?? Connectivity();

  final NetworkReachability _reachability;
  final Connectivity _connectivity;
  StreamSubscription<List<ConnectivityResult>>? _subscription;
  Timer? _debounce;

  void start() {
    WidgetsBinding.instance.addObserver(this);
    _subscription = _connectivity.onConnectivityChanged.listen((_) {
      _debounce?.cancel();
      _debounce = Timer(const Duration(milliseconds: 350), check);
    });
    check();
  }

  Future<void> check() => _reachability.check().then(updateState);

  @override
  void didChangeAppLifecycleState(AppLifecycleState state) {
    if (state == AppLifecycleState.resumed) check();
  }

  void updateState(NetworkState state) {
    // Publish to Riverpod, Bloc, ChangeNotifier, or the view layer.
  }

  Future<void> dispose() async {
    WidgetsBinding.instance.removeObserver(this);
    _debounce?.cancel();
    await _subscription?.cancel();
    _reachability.dispose();
  }
}
```

In a real controller, inject the same `Connectivity` instance into both the controller and `NetworkReachability`. With `connectivity_plus` 7.x, `checkConnectivity()` and `onConnectivityChanged` use `List<ConnectivityResult>`, so older examples that compare one result directly need migration. A list can contain more than one transport, and `none` should not be treated as the only possible “not usable” state.

## What to test before shipping

Use a fake `http.Client` and a fake `Connectivity` rather than relying on a real network in unit tests. The useful cases are decisions, not latency benchmarks.

- Wi-Fi plus a `204` health response → `reachable`.
- Wi-Fi plus a captive portal or DNS failure → request failure, not a false green state.
- Mobile plus a `401` health response → `reachable` if the endpoint requires auth.
- Wi-Fi plus a `503` response → `serverError`, while the transport indicator can remain visible.
- `[ConnectivityResult.none]` on launch → do not wait for the stream; show the initial no-transport state.
- Resume after backgrounding → call `check()` even if no stream event arrived.
- Repeated reconnect events → one debounced probe and no leaked subscriptions.

Do not probe `google.com` or another unrelated public host to decide whether your API works. That creates a different dependency, can violate captive-portal assumptions, and can report “online” while your own API is blocked. `connectivity_plus` remains valuable, but its output belongs in the transport layer. The request boundary owns the final decision for the operation that matters.

References: [connectivity_plus package documentation](https://pub.dev/packages/connectivity_plus), [connectivity_plus Android background behavior](https://pub.dev/packages/connectivity_plus#android), and [the plugin issue about transport results not proving internet access](https://github.com/fluttercommunity/plus_plugins/issues/632).
