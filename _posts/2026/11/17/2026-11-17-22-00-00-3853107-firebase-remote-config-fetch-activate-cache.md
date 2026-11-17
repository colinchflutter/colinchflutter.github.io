---
layout: post
title: "firebase_remote_config Fetch and Activate - Avoid Stale Flutter Flags Without Blocking Startup"
description: "Use firebase_remote_config fetch, activate, and minimumFetchInterval deliberately so Flutter feature flags stay predictable without a slow startup or fetch throttling."
date: 2026-11-17
tags: [firebase_remote_config, networking, caching, performance, Flutter]
comments: true
share: true
---

![firebase_remote_config fetch and activate cache flow](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

If a Flutter app uses `firebase_remote_config` for feature flags, the safest production default is to activate the last known values immediately, then fetch new values for the next session. That choice fits UI experiments, copy changes, and gradual rollouts. It does not fit a kill switch that must take effect within seconds; that needs a separate real-time or server-side control path.

## The trap: fetch does not mean visible

Remote Config has three different states. Defaults protect a first launch, fetched values sit in local storage, and activated values are what getters return to the app. Calling `fetch()` alone can therefore succeed while the screen keeps using the previous configuration. Calling `fetchAndActivate()` on every launch solves that confusion, but can also make startup wait for the network and encourages an overly short cache interval during testing.

| Requirement | Better policy | Cost or limitation |
| --- | --- | --- |
| Render the app quickly | `activate()` cached values, then `fetch()` | New values wait for a later activation |
| Apply a harmless copy change now | `fetchAndActivate()` after the screen is ready | The current session can change mid-flow |
| Emergency flag with low latency | Real-time Remote Config or server decision | More lifecycle and fallback handling |
| Offline first launch | `setDefaults()` before reads | Defaults must be maintained with the app |

Firebase documents a 12-hour default minimum fetch interval. A call made inside that window can be served from local cache, so “I published a new value but the app did not change” is often expected behavior rather than a broken Flutter plugin. Repeated fetches can also report `RemoteConfigFetchStatus.throttle`.

## A reproducible startup pattern

The following repository makes the timing explicit. It uses a 12-hour production interval, activates the previous session before building the first screen, and fetches in the background without activating during the current session.

```dart
import 'dart:async';

import 'package:firebase_core/firebase_core.dart';
import 'package:firebase_remote_config/firebase_remote_config.dart';

class AppRemoteConfig {
  AppRemoteConfig(this._remoteConfig);

  final FirebaseRemoteConfig _remoteConfig;

  Future<void> initialize() async {
    await _remoteConfig.setConfigSettings(
      RemoteConfigSettings(
        fetchTimeout: const Duration(seconds: 8),
        minimumFetchInterval: const Duration(hours: 12),
      ),
    );

    await _remoteConfig.setDefaults(const {
      'checkout_redesign_enabled': false,
      'support_email': 'support@example.com',
    });

    // Make the last successfully fetched configuration available now.
    await _remoteConfig.activate();

    // Refresh the cache for the next session. Do not block first paint.
    unawaited(_fetchForNextSession());
  }

  bool get checkoutRedesignEnabled =>
      _remoteConfig.getBool('checkout_redesign_enabled');

  String get supportEmail => _remoteConfig.getString('support_email');

  Future<void> _fetchForNextSession() async {
    try {
      await _remoteConfig.fetch();
      // Deliberately do not call activate() here.
      // The next initialize() call applies this fetched snapshot.
    } on FirebaseException catch (error) {
      // Keep the activated snapshot and record the error for diagnostics.
      // A fetch failure must not replace a usable local configuration.
      unawaited(_recordFetchFailure(error.code));
    }
  }

  Future<void> _recordFetchFailure(String code) async {
    // Send a rate-limited metric or structured log in the real app.
  }
}
```

Initialize the repository after `Firebase.initializeApp()` and before creating widgets that read the flags. The defaults are not a secret store: Remote Config keys and values can be inspected by the client, so do not put API keys, credentials, or authorization decisions there.

The important detail is the separation between `fetch()` and `activate()`. A release can fetch a newly published value while the user is halfway through checkout, but the checkout screen continues with one consistent snapshot. On the next cold start, `activate()` applies the cached result before the first meaningful frame.

## When `fetchAndActivate()` is the right choice

Use `fetchAndActivate()` only when changing the value during the current session is safe and the UI is ready to handle it. A language hint, a non-critical banner, or a cosmetic experiment may fit. A navigation tree, payment flow, or permission gate usually benefits from a stable snapshot instead.

For a current-session update, keep the activation boundary visible rather than hiding it inside a generic service:

```dart
Future<bool> refreshVisibleConfig() async {
  try {
    return await FirebaseRemoteConfig.instance.fetchAndActivate();
  } on FirebaseException catch (error) {
    // Continue with the last activated values on timeout or throttle.
    // Log error.code and lastFetchStatus, but do not clear defaults here.
    return false;
  }
}
```

This method returns whether the call activated changed values. It does not prove that the backend was contacted; the minimum fetch interval still applies. For development, a short interval can help iteration, but it should not be copied into production. A low interval across a large test population increases fetch volume and makes throttling harder to distinguish from an application bug.

## Verification checklist

- Set defaults and verify the app has a usable state with airplane mode enabled.
- Log `lastFetchStatus` and `lastFetchTime` during a debug build; distinguish `success`, `failure`, and `throttle`.
- Publish a changed parameter, then verify the expected cache window before declaring the rollout broken.
- Kill and relaunch the app to confirm that a fetched-but-not-yet-activated value is applied at the intended boundary.
- Reopen the screen while a fetch is running; it should keep the same activated snapshot.
- Test a malformed or missing parameter and preserve a typed default rather than assuming a non-null backend value.

For a flag that must bypass the minimum interval, Firebase's real-time Remote Config can notify a connected app and fetch an update. That feature is not available on Web, and it still requires a decision about when to call `activate()`. “Fetched immediately” and “safe to change the current screen” are separate requirements.

The practical rule is simple: treat Remote Config as a cached configuration source, not as a synchronous database. Define defaults, choose an activation boundary, keep the cache interval intentional, and make failures fall back to the last known good snapshot.

Sources checked 2026-11-17: [Firebase Remote Config for Flutter](https://firebase.google.com/docs/remote-config/flutter/get-started), [Remote Config loading strategies](https://firebase.google.com/docs/remote-config/loading), and [`firebase_remote_config` API documentation](https://pub.dev/documentation/firebase_remote_config/latest/firebase_remote_config/).
