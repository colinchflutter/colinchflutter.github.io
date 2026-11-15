---
layout: post
title: "connectivity_plus Reachability - Stop Treating Wi-Fi as a Successful API Request"
description: "Use connectivity_plus as a retry hint, not an Internet guarantee. This Flutter pattern handles Android resume checks, iOS duplicate events, captive portals, and safe request retries."
date: 2026-11-16
tags: [connectivity_plus, networking, caching, Android, iOS, testing, Flutter]
comments: true
share: true
---

![Flutter connectivity_plus network signal and API reachability boundary](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?auto=format&fit=crop&w=1600&q=80)

The safe choice is to use `connectivity_plus` as a **reason to attempt a request**, not as proof that the request will succeed. This fits an app that needs offline banners, sync triggers, or a resume-time refresh. It does not fit code that skips every HTTP request when the result is `none`, or code that treats `wifi` as “the API is reachable”. A captive portal, broken router, VPN route, DNS failure, or server outage can make the network type and the application result disagree.

## The failure is a policy mismatch

The package documentation explicitly says that a connection type does not guarantee Internet access. It also documents platform-specific behavior that is easy to miss in a clean emulator run:

| Situation | What `connectivity_plus` can tell you | What the app still has to decide |
| --- | --- | --- |
| Wi-Fi with a captive portal | A Wi-Fi transport exists | Whether the API request can authenticate and complete |
| Android app resumes after backgrounding | The old event may be stale | Re-check the current state and retry eligible work |
| iOS reconnects | `none` can be followed by `wifi` | Debounce the sync trigger and let the request decide |
| VPN-only route | A result may not describe the usable route | Perform an actual reachability or API check |

The production bug usually looks like this: the app receives `wifi`, immediately starts a large sync, then shows a generic success state because the connectivity callback was mistaken for a response. The opposite bug is also possible. A VPN can provide working Internet access while `checkConnectivity()` reports no usable type, so a hard gate prevents a request that could have succeeded.

## Keep the package at the edge of the network policy

The feature layer should not read `Connectivity().checkConnectivity()` directly. Put the package behind a small coordinator that emits a request hint. The HTTP client remains the authority for success, timeout, authentication, and server errors.

This example intentionally makes the request injectable. It can be replaced with a repository call or a client health endpoint without coupling the rest of the app to a plugin stream.

```dart
import 'dart:async';

import 'package:connectivity_plus/connectivity_plus.dart';

class NetworkSyncCoordinator {
  NetworkSyncCoordinator({
    Connectivity? connectivity,
    required Future<void> Function() sync,
  })  : _connectivity = connectivity ?? Connectivity(),
        _sync = sync;

  final Connectivity _connectivity;
  final Future<void> Function() _sync;
  StreamSubscription<List<ConnectivityResult>>? _subscription;
  Timer? _debounce;
  bool _running = false;

  Future<void> start() async {
    await _attempt('startup');
    _subscription = _connectivity.onConnectivityChanged.listen((_) {
      _debounce?.cancel();
      _debounce = Timer(const Duration(milliseconds: 500), () {
        unawaited(_attempt('connectivity-change'));
      });
    });
  }

  Future<void> onResumed() => _attempt('resume');

  Future<void> _attempt(String reason) async {
    if (_running) return;
    _running = true;
    try {
      // This is only a hint. The sync call still handles timeout and errors.
      final result = await _connectivity.checkConnectivity();
      if (result.contains(ConnectivityResult.none) && reason != 'resume') {
        return;
      }
      await _sync();
    } finally {
      _running = false;
    }
  }

  Future<void> dispose() async {
    _debounce?.cancel();
    await _subscription?.cancel();
  }
}
```

The `none` branch above is deliberately conservative only for event-driven attempts. It still permits the explicit resume check to probe the real endpoint. If the product cannot tolerate a missed sync, remove the branch and let the request layer apply its normal timeout and retry policy. The important boundary is that a positive connectivity result never marks data as synchronized.

## Android and iOS need different triggers

On Android, the package notes that background connectivity broadcasts are not delivered in the same way from Android O onward. A listener that was correct while the app was visible can miss the transition that happened while it was paused. Call `onResumed()` from an app-lifecycle observer and make it idempotent.

On iOS and macOS, `NWPathMonitor` may emit multiple or unreliable results around a reconnect, including `none` followed immediately by `wifi`. The 500 ms debounce is not a reachability test; it only prevents two identical sync jobs from starting during one transition. The request itself must still be deduplicated by the repository or server.

For a write operation, add an idempotency key or a client operation ID. A lost response is not proof that the server rejected the write. Retrying a payment, upload, or order creation solely because connectivity changed can duplicate the side effect.

## Do not turn a transport signal into a data state

Use separate states so the UI does not claim more than the evidence supports:

| State | Evidence | UI meaning |
| --- | --- | --- |
| `networkHintAvailable` | A transport type was reported | “Trying to sync” |
| `requestSucceeded` | The API returned an accepted response | “Synced” |
| `requestFailed` | Timeout, DNS, HTTP, or parse error | “Sync failed; retry available” |
| `offlineConfirmed` | An actual probe or request failed for a defined reason | “Offline or unreachable” |

The package’s own `hasConnectivity` convenience API does not change this rule. It answers a transport question, not whether your API, DNS resolver, authentication gateway, or captive-portal flow is healthy.

## A small release checklist

- Call `checkConnectivity()` when the app starts and resumes; do not depend only on a background event.
- Debounce reconnect signals, then guard the sync operation against concurrent execution.
- Treat the HTTP response as the source of truth for successful synchronization.
- Test Wi-Fi without Internet, captive-portal Wi-Fi, VPN-only access, airplane-mode transitions, and server timeouts.
- Cancel the stream subscription when the owner is disposed; otherwise route changes can create duplicate sync listeners.
- Give writes an idempotency key before enabling automatic retries.
- If upgrading to `connectivity_plus` 7.x, review the stated requirements: Flutter 3.19+, Dart 3.3+, Java 17, and Android Gradle Plugin 8.12.1 or newer as checked on 2026-11-16.

`connectivity_plus` is useful when its result stays a hint in the architecture. Let it wake a bounded attempt, let the network client report what actually happened, and let the data layer decide whether repeating the operation is safe. That separation covers captive portals and flaky reconnect events without pretending that a green Wi-Fi icon is a successful backend call.

Sources: [connectivity_plus README and platform notes](https://pub.dev/packages/connectivity_plus), [connectivity_plus changelog](https://pub.dev/packages/connectivity_plus/changelog), and [VPN-related connectivity issue #3810](https://github.com/fluttercommunity/plus_plugins/issues/3810) (checked 2026-11-16).
