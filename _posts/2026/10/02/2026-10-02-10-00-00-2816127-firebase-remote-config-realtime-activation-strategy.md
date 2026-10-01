---
layout: post
title: "firebase_remote_config Real-time Updates - Avoid Stale Flags and Mid-screen UI Breaks"
description: "A production-safe Flutter pattern for firebase_remote_config real-time updates: choose fetchAndActivate timing, activate safely, and handle Web and throttling limits."
date: 2026-10-02
tags: [firebase_remote_config, Flutter, Firebase, networking, Web, Android, iOS]
comments: true
share: true
---

![Firebase Remote Config real-time update flow](https://firebase.google.com/static/images/products/icons/remote-config.svg)

*The key detail is the boundary between fetching a new value and activating it in the visible UI.*

If a Flutter app uses `firebase_remote_config` for feature flags, staged rollouts, or emergency switches, the safest default is **activate cached values at startup, fetch in the background, and activate real-time changes at a deliberate UI boundary**. A small banner or server-driven copy can update immediately; a checkout layout or navigation rule should not change while the user is interacting with it. This pattern does not apply to secrets—Remote Config values are readable by the client.

## The production failure is usually timing

`fetch()` downloads and caches values. `activate()` makes the latest fetched values visible to getters. `fetchAndActivate()` does both, which is convenient but can make startup dependent on a network request. Real-time listening does not replace the startup fetch: it keeps a foreground HTTP connection and automatically fetches when Firebase publishes a newer template. On Web, real-time Remote Config is not available.

| Requirement | Recommended flow | Trade-off |
| --- | --- | --- |
| Stable startup and changes on next launch | `activate()` → `fetch()` | A published flag may wait until the next launch |
| Small non-critical UI values | `fetchAndActivate()` on launch | Visible layout can change after loading |
| Fast rollback during a session | `onConfigUpdated` → selective `activate()` | Needs a UI-safe activation boundary |
| Flutter Web | `fetch()` with defaults | No real-time listener |

## A single service owns the lifecycle

Keep Remote Config behind one service so screens do not independently fetch, activate, and subscribe. The following example uses the current `firebase_remote_config` API and activates a remote update only when the app has agreed that the affected UI is safe to refresh.

```dart
import 'dart:async';

import 'package:firebase_remote_config/firebase_remote_config.dart';

class AppRemoteConfig {
  AppRemoteConfig(this._config);

  final FirebaseRemoteConfig _config;
  StreamSubscription<RemoteConfigUpdate>? _updates;

  Future<void> start() async {
    await _config.setDefaults(const {
      'new_checkout_enabled': false,
      'promo_banner_text': '',
    });

    // Use a production-sized interval; do not ship a five-minute dev value.
    await _config.setConfigSettings(RemoteConfigSettings(
      fetchTimeout: const Duration(seconds: 10),
      minimumFetchInterval: const Duration(hours: 12),
    ));

    // Make a previously fetched value available without blocking the first frame.
    await _config.activate();
    unawaited(_config.fetch());

    _updates = _config.onConfigUpdated.listen((update) async {
      final changed = update.updatedKeys;
      if (changed.contains('promo_banner_text')) {
        await _config.activate();
        // Notify the banner model here, not an arbitrary widget.
      }
    });
  }

  bool get checkoutEnabled => _config.getBool('new_checkout_enabled');

  Future<void> dispose() => _updates?.cancel() ?? Future<void>.value();
}
```

For a feature that changes navigation, pricing, or a form, collect the update first and activate it on the next route entry or next app launch. `updatedKeys` is useful for avoiding a full rebuild when an unrelated parameter changed. Also guard the fetch: defaults must keep the app usable offline, and a fetch failure should not clear the last activated value.

## Checklist before shipping

- Set defaults for every key read by Dart. Treat them as a working offline configuration, not placeholders.
- Test cold start with no network, a timed-out fetch, and a previously fetched but not activated value.
- Test Android and iOS foreground/background transitions. The real-time connection is maintained only in the foreground and restarts after returning.
- Keep the production `minimumFetchInterval` conservative. Repeated polling can be throttled; real-time updates bypass that interval when supported.
- Verify Web separately. Do not call a real-time-only assumption from shared code.
- Log `lastFetchStatus` and `lastFetchTime`, but never put tokens or private policy data in Remote Config.

The practical rule is simple: fetch often enough for freshness, activate only when the current UI can tolerate change, and keep a valid local default for every failure path. That turns `firebase_remote_config` from a hidden source of inconsistent screens into an explicit configuration pipeline.

Sources: [Firebase real-time Remote Config](https://firebase.google.com/docs/remote-config/flutter/real-time), [Flutter Remote Config setup](https://firebase.google.com/docs/remote-config/flutter/get-started), [Remote Config loading strategies](https://firebase.google.com/docs/remote-config/loading), [firebase_remote_config API](https://pub.dev/documentation/firebase_remote_config/latest/firebase_remote_config/FirebaseRemoteConfig-class.html)
