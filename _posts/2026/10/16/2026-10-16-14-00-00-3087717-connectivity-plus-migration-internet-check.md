---
layout: post
title: "connectivity_plus 6 Migration - Stop Treating Flutter Network Type as Internet Access"
description: "Migrate connectivity_plus to List<ConnectivityResult> and separate transport status from a real internet check, with lifecycle-safe Flutter code."
date: 2026-10-16
tags: [connectivity_plus, networking, migration, Android, iOS, testing, Flutter]
comments: true
share: true
---

![Flutter connectivity_plus migration from a network type event to an internet health check](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?auto=format&fit=crop&w=1600&q=80)

If an existing Flutter app still expects one `ConnectivityResult`, migrate `connectivity_plus` to `List<ConnectivityResult>` first, then keep “a network interface exists” separate from “the API is reachable”. This is useful for offline banners and sync scheduling. It is not enough for payment, upload, or retry decisions, because Wi-Fi can be connected to a captive portal or a broken router.

## The migration changes two assumptions

`connectivity_plus` 6.0.1 changed the result to support multiple simultaneous connection types. The current API returns a list from both `checkConnectivity()` and `onConnectivityChanged`. A direct type replacement compiles only after the business logic is updated too.

| Old assumption | Safer current meaning |
| --- | --- |
| `result == ConnectivityResult.wifi` | `types.contains(ConnectivityResult.wifi)` |
| One active type | Several types may be reported |
| `wifi` means online | `wifi` means a transport is available |
| Stream event is the whole truth | Recheck on resume and verify the API when needed |

The second assumption is the production trap. The package documentation explicitly says that a connection type does not guarantee internet access. Android also stops delivering connectivity changes to background apps from Android O, so a value captured before the app was paused can be stale.

## Normalize the new API at one boundary

Keep package-specific types out of feature widgets. This adapter makes the migration visible and gives tests one small surface.

```dart
import 'dart:async';
import 'package:connectivity_plus/connectivity_plus.dart';

class TransportMonitor {
  TransportMonitor(this._connectivity);

  final Connectivity _connectivity;
  StreamSubscription<List<ConnectivityResult>>? _subscription;

  Future<Set<ConnectivityResult>> readTypes() async {
    return (await _connectivity.checkConnectivity()).toSet();
  }

  void start(void Function(Set<ConnectivityResult>) onChange) {
    _subscription = _connectivity.onConnectivityChanged
        .map((types) => types.toSet())
        .distinct()
        .listen(onChange);
  }

  Future<void> dispose() => _subscription?.cancel() ?? Future.value();
}
```

The `distinct()` call is deliberate. iOS `NWPathMonitor` can report a transient `none` followed by `wifi` during reconnection. Treating every event as a command can start duplicate sync jobs. The adapter reports transport state; it does not label the user online.

## Add a real reachability check only where it matters

For an API-backed action, use a small health endpoint owned by the same backend. Do not probe an arbitrary public URL: a successful captive-portal response or a blocked domain can give the wrong product answer.

```dart
import 'package:http/http.dart' as http;

Future<bool> apiIsReachable(Uri healthUri) async {
  try {
    final response = await http
        .get(healthUri)
        .timeout(const Duration(seconds: 3));
    return response.statusCode == 200;
  } on Exception {
    return false;
  }
}
```

Use transport events to refresh the UI quickly, then verify reachability immediately before a critical sync. On `AppLifecycleState.resumed`, call `readTypes()` again instead of trusting the last stream event. A failed probe should be treated as a retryable network result only when the request is safe to repeat; a POST that may already have reached the server needs an idempotency key or an explicit user retry.

## A migration checklist that catches the real break

- [ ] Change callbacks and mocks from `ConnectivityResult` to `List<ConnectivityResult>`.
- [ ] Replace equality checks with `contains` or a normalized set.
- [ ] Test Wi-Fi plus mobile, `none`, and duplicate reconnect events.
- [ ] Recheck after Android resume; do not run background-only assumptions from the stream.
- [ ] Keep the health endpoint check separate from the package adapter.
- [ ] Test a captive portal or a server timeout if the feature makes writes.
- [ ] Pin the resolved package version in `pubspec.lock` before changing the migration.

The practical boundary is simple: `connectivity_plus` tells the app which transports the operating system can see. A backend probe tells the app whether this particular operation can reach its server. Keeping those answers separate fixes the 6.x type migration without turning a network icon into a false promise of online work.

Sources: [connectivity_plus package documentation](https://pub.dev/packages/connectivity_plus), [connectivity_plus changelog](https://pub.dev/packages/connectivity_plus/changelog), and [Flutter offline-first guidance](https://docs.flutter.dev/app-architecture/design-patterns/offline-first).
