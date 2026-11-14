---
layout: post
title: "firebase_messaging Notification Taps - Prevent Duplicate Flutter Navigation After Cold Start"
description: "Handle firebase_messaging notification taps from terminated and background states without duplicate navigation, lost routes, or a race with Flutter startup."
date: 2026-11-14
tags: [firebase_messaging, notifications, navigation, Flutter, Android, iOS]
comments: true
share: true
---

![Firebase Cloud Messaging notification tap flow from a terminated Flutter app](https://images.unsplash.com/photo-1551650975-87deedd944c3?auto=format&fit=crop&w=1600&q=80)

*The important boundary is one idempotent tap handler shared by both launch paths, not a second navigation call in `main.dart`.*

If a Flutter app opens a chat, order, or document from a push notification, handle both `getInitialMessage()` and `onMessageOpenedApp`, then deduplicate before navigation. This fits apps where the destination matters after a cold start or background resume. It is unnecessary for notifications that only display text and never navigate. Firebase documents the two lifecycle paths separately, so wiring only one of them guarantees a missing case.

## Why the obvious example breaks in production

The official example calls `_handleMessage` for the initial message and also subscribes to `onMessageOpenedApp`. That is a good starting point, but production startup adds two traps:

| Situation | Typical symptom | Safer boundary |
| --- | --- | --- |
| App terminated | route is requested before the router is ready | queue until navigation is ready |
| App in background | the same intent is observed by more than one startup layer | dedupe by message id |
| Older plugin/device combination | `getInitialMessage()` is returned again | make handling idempotent |
| Message has no id | two legitimate taps look identical | use a documented fallback key and accept its limit |

The package changelog has fixes for terminated-tap delivery and startup races, but an app should still make its own navigation side effect safe. A GitHub report also shows why “the SDK consumes it exactly once” should not be the only guard: repeated `getInitialMessage()` results have been reproduced on real devices.

## Use one queue and one dedupe set

The following controller keeps the package callbacks thin. Call `markReady` after the app router has been mounted; call `dispose` with the owning widget or service.

```dart
import 'dart:async';

import 'package:firebase_messaging/firebase_messaging.dart';

class NotificationTapController {
  final _pending = <RemoteMessage>[];
  final _handledKeys = <String>{};
  StreamSubscription<RemoteMessage>? _subscription;
  void Function(RemoteMessage message)? _navigate;
  bool _ready = false;

  Future<void> start(void Function(RemoteMessage) navigate) async {
    _navigate = navigate;

    // Install the stream listener before awaiting startup work.
    _subscription = FirebaseMessaging.onMessageOpenedApp.listen(_accept);

    final initial = await FirebaseMessaging.instance.getInitialMessage();
    if (initial != null) _accept(initial);
  }

  void markReady() {
    _ready = true;
    for (final message in List<RemoteMessage>.from(_pending)) {
      _navigate?.call(message);
    }
    _pending.clear();
  }

  void _accept(RemoteMessage message) {
    final key = message.messageId ?? _fallbackKey(message);
    if (!_handledKeys.add(key)) return;

    if (_ready) {
      _navigate?.call(message);
    } else {
      _pending.add(message);
    }
  }

  String _fallbackKey(RemoteMessage message) {
    final data = message.data;
    return '${message.sentTime?.millisecondsSinceEpoch}:'
        '${data['type']}:${data['id']}';
  }

  Future<void> dispose() async => _subscription?.cancel();
}
```

The controller is deliberately not a permanent event log. If the process dies before `markReady`, the next launch may receive the same message again; the destination must therefore be safe to open twice, and the server-side action must be idempotent too. For a stable fallback key, put an explicit `event_id` in every data payload rather than hashing display text.

## Startup checklist

- Register the `onMessageOpenedApp` listener before awaiting other initialization.
- Call `getInitialMessage()` once from the owner of the tap controller.
- Do not navigate until `MaterialApp` or `GoRouter` has a mounted context.
- Use `messageId` or a server-generated `event_id`; do not dedupe on notification title.
- Test terminated, background, foreground, rapid taps, and a process restart in release mode.
- Pin and verify the plugin version when upgrading; recent releases changed iOS and Android launch delivery behavior.

The practical rule is simple: Firebase owns delivery, while the app owns idempotency and route readiness. That separation prevents a notification tap from becoming either two pushes onto the stack or no navigation at all.

Sources: [Firebase notification interaction guide](https://firebase.google.com/docs/cloud-messaging/flutter/receive#handling_interaction), [firebase_messaging changelog](https://pub.dev/packages/firebase_messaging/changelog), and [FlutterFire issue #10768](https://github.com/firebase/flutterfire/issues/10768).
