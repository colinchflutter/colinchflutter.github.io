---
layout: post
title: "firebase_messaging Notification Tap Deduplication - Fix Flutter Double Navigation After Android Termination"
description: "Stop firebase_messaging from opening the same Flutter route twice by separating terminated and background taps, upgrading safely, and deduplicating message IDs."
date: 2026-10-01
tags: [firebase_messaging, navigation, Android, iOS, debugging]
comments: true
share: true
---

![firebase_messaging notification tap deduplication flow](/assets/images/firebase-messaging-notification-tap-deduplication.png)

The safest fix for a Flutter app that navigates twice after a notification tap is to treat `getInitialMessage()` and `onMessageOpenedApp` as two delivery paths for one user action, then make the route handler idempotent. This applies to apps supporting terminated and background states; it does not replace notification permission or platform setup. On Android, `firebase_messaging` 16.7.0 also includes a fix to skip `onMessageOpenedApp` for terminated notification taps, but application-level deduplication is still useful during upgrades and on other platforms.

## Why the same tap reaches two callbacks

`getInitialMessage()` is for an app opened from a terminated state. `onMessageOpenedApp` is a stream for an app brought forward from the background. Firebase's Flutter guide describes them as separate states, but older Android plugin versions could expose a terminated tap through both paths. If both callbacks call `Navigator.push`, the user sees two copies of the detail page or a route stack that immediately pops back to the wrong place.

| App state when the user taps | Primary source | Handler rule |
|---|---|---|
| Terminated | `getInitialMessage()` | Handle once after the router is ready |
| Background | `onMessageOpenedApp` | Handle stream events |
| Foreground | `onMessage` | Show in-app UI; do not treat as a tap |
| Any state during migration | Either path | Reject an already handled event |

The key mistake is deciding “which callback is correct” inside the page. Deduplicate at the notification boundary before navigation, where the message ID and app lifecycle state are visible.

## A single idempotent entry point

The following service subscribes before awaiting the initial message, then sends both paths through one key check. This ordering also gives the stream a chance to receive an event while startup is still waiting for Firebase or local storage.

```dart
import 'dart:async';

import 'package:firebase_messaging/firebase_messaging.dart';

class NotificationTapRouter {
  NotificationTapRouter(this.onTap);

  final void Function(RemoteMessage message) onTap;
  StreamSubscription<RemoteMessage>? _subscription;
  final Set<String> _handled = <String>{};

  Future<void> start() async {
    _subscription = FirebaseMessaging.onMessageOpenedApp.listen(_accept);

    final initial =
        await FirebaseMessaging.instance.getInitialMessage();
    if (initial != null) {
      _accept(initial);
    }
  }

  void _accept(RemoteMessage message) {
    final key = message.messageId ?? _fallbackKey(message);
    if (!_handled.add(key)) {
      return;
    }
    onTap(message);
  }

  String _fallbackKey(RemoteMessage message) {
    final data = message.data.entries.toList()
      ..sort((a, b) => a.key.compareTo(b.key));
    return '${message.sentTime?.millisecondsSinceEpoch}:${data.join('&')}';
  }

  Future<void> dispose() async {
    await _subscription?.cancel();
  }
}
```

Call `start()` once from an application-scoped object, not from every page that can receive a deep link. The router must also be ready before `onTap` navigates. If startup navigation is asynchronous, queue the `RemoteMessage` until the `GoRouter` or `NavigatorState` has a valid context rather than calling `push` from a background initialization callback.

## Upgrade decision and reproducible checks

As of October 1, 2026, the package changelog lists `firebase_messaging` 16.7.0 and an Android fix for skipping `onMessageOpenedApp` on terminated notification taps. Upgrade the FlutterFire packages together when the project's Firebase SDK constraints permit it, then keep the idempotent handler for mixed-version rollout and iOS behavior.

```yaml
dependencies:
  firebase_messaging: ^16.7.0
```

Run the same test matrix after the upgrade:

- Force-stop the Android app, tap a notification, and verify one navigation event.
- Put the app in the background, tap a notification, and verify only the stream path navigates.
- Leave the app foregrounded and send a notification; verify `onMessage` does not push the detail route.
- Repeat a tap with a missing `messageId`; verify the sorted-data fallback key prevents a duplicate.
- Cold-start twice with the same notification payload; verify the process-local guard resets only when a new app session is intended.

The in-memory `Set` protects one process session. If tapping a notification can retry an order, payment, or other one-time operation, also make the server endpoint idempotent. A navigation guard can stop duplicate screens, but it cannot undo a duplicate backend write.

In short, separate the three lifecycle paths, centralize tap handling, deduplicate before navigation, and upgrade to the plugin version containing the terminated-tap fix. The code still solves the underlying integration problem when a release cannot move to the newest FlutterFire version immediately.

Sources: [Firebase Flutter notification interaction guide](https://firebase.google.com/docs/cloud-messaging/flutter/receive-messages), [firebase_messaging changelog](https://pub.dev/packages/firebase_messaging/changelog).
