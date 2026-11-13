---
layout: post
title: "web_socket_client Lifecycle Reconnect - Keep Flutter WebSockets from Dying After Resume"
description: "Use web_socket_client with Flutter app lifecycle events to stop dead sockets in the background, recreate them on resume, and ignore stale callbacks from an older connection."
date: 2026-11-14
tags: [web_socket_client, networking, Flutter, Android, iOS, Web]
comments: true
share: true
---

![Flutter WebSocket lifecycle reconnect flow](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?auto=format&fit=crop&w=1600&q=80)

*The important boundary is not the reconnect delay; it is replacing the old socket when the Flutter app becomes active again.*

`web_socket_client` is a good fit for a foreground realtime feature that should reconnect after a brief network failure. It is not enough to create one global socket and trust it through an Android or iOS background transition. For chat presence, live dashboards, or a trading screen that is only meaningful while visible, close the client when the app is hidden and create a new client on resume. A background sync job needs a different design because Flutter does not guarantee every lifecycle notification before the process is killed.

## Why a connected socket can be unusable after resume

`web_socket_client` automatically reconnects when its connection is disrupted, using a backoff strategy. That helps with a network flap, but `close()` is a deliberate shutdown: the client enters `disconnected` and will not reconnect itself. The practical consequence is that a lifecycle handler must create a new `WebSocket` instance after a deliberate close.

| Situation | Keep the same client? | Action |
| --- | ---: | --- |
| Short network failure while app is visible | Yes | Let the package back off and reconnect |
| App enters `hidden` or `paused` | No | Close and discard the client |
| App returns to `resumed` | No | Create a client and attach fresh listeners |
| Logout or screen disposal | No | Close permanently; do not schedule reconnect |

Flutter documents `resumed`, `hidden`, and `paused` as different states, and warns that applications should not rely on receiving every notification. The code below handles the normal foreground/background path, while the server must still treat every connection as a new session.

## Make the socket generation explicit

The generation number prevents a callback from an old socket from changing the state of a newer one. This matters when a close, a resume event, and a delayed connection notification arrive close together.

```dart
import 'dart:async';

import 'package:flutter/widgets.dart';
import 'package:web_socket_client/web_socket_client.dart';

class ForegroundSocketController {
  ForegroundSocketController(this.uri) {
    _lifecycle = AppLifecycleListener(onStateChange: _onLifecycleState);
  }

  final Uri uri;
  late final AppLifecycleListener _lifecycle;
  WebSocket? _socket;
  StreamSubscription<Object?>? _messages;
  StreamSubscription<Object?>? _connection;
  int _generation = 0;
  bool _disposed = false;
  bool _shouldRun = true;

  void _onLifecycleState(AppLifecycleState state) {
    if (state == AppLifecycleState.hidden ||
        state == AppLifecycleState.paused) {
      _shouldRun = false;
      _replaceSocket(null);
    } else if (state == AppLifecycleState.resumed) {
      _shouldRun = true;
      _connect();
    }
  }

  void _connect() {
    if (_disposed || !_shouldRun || _socket != null) return;

    final int generation = ++_generation;
    final socket = WebSocket(uri, timeout: const Duration(seconds: 10));
    _socket = socket;

    _messages = socket.messages.listen(
      (message) {
        if (generation != _generation || _socket != socket) return;
        _handleMessage(message);
      },
      onError: (Object error, StackTrace stack) {
        if (generation == _generation) _handleError(error, stack);
      },
    );

    _connection = socket.connection.listen((state) {
      if (generation != _generation || _socket != socket) return;
      _handleConnectionState(state);
    });
  }

  void _replaceSocket(WebSocket? next) {
    ++_generation;
    final old = _socket;
    _socket = next;
    unawaited(_messages?.cancel());
    unawaited(_connection?.cancel());
    _messages = null;
    _connection = null;
    old?.close();
  }

  void send(Object message) {
    final socket = _socket;
    if (socket == null) return; // Queue only idempotent messages if needed.
    socket.send(message);
  }

  void dispose() {
    if (_disposed) return;
    _disposed = true;
    _shouldRun = false;
    _lifecycle.dispose();
    _replaceSocket(null);
  }

  void _handleMessage(Object message) {}
  void _handleConnectionState(Object state) {}
  void _handleError(Object error, StackTrace stack) {}
}
```

The controller owns the socket instead of a screen widget. A view can subscribe to the controller’s state, but it should not call `WebSocket()` from `build()` or recreate the connection on every provider rebuild. `send()` deliberately drops messages while disconnected. Queuing is safe only when the protocol has an idempotency key or a server acknowledgement; blindly replaying a payment, edit, or presence event is worse than showing an offline state.

## `web_socket_client` versus `web_socket_channel`

`web_socket_channel` exposes a cross-platform `WebSocketChannel`, its single-subscription `stream`, and a `ready` future, but it does not provide the automatic reconnect state machine shown above. If you use it, a reconnect must build a new channel, attach a new stream listener, and invalidate callbacks from the old channel. The same generation rule still applies; changing packages does not remove the race.

Do not use `connectivity_plus` as the proof that the socket is usable. Network type changes can trigger a reconnect attempt, but only the socket handshake and your application-level authentication confirm that the realtime channel works. On `reconnected`, refresh subscriptions or request a cursor from the server if events may have been missed.

## Release checklist

- Open the feature, background the app, wait longer than the server idle timeout, then resume.
- Toggle Wi-Fi and cellular while the app remains visible; verify backoff does not create parallel clients.
- Log out during `reconnecting`; confirm no later callback restores the authenticated session.
- Resume after a process restart and verify the initial snapshot is fetched again.
- Test Web separately: a browser tab can be hidden without behaving like a mobile process pause.
- Add a server-side session id and generation to logs so duplicate connections are diagnosable.

The safe default is simple: let `web_socket_client` recover transient failures while the app is visible, but treat a lifecycle-driven close as terminal for that object. Recreate the client on resume, attach fresh listeners, and reject stale callbacks by identity or generation.

Sources: [web_socket_client package documentation](https://pub.dev/packages/web_socket_client), [web_socket_channel API](https://pub.dev/documentation/web_socket_channel/latest/web_socket_channel/WebSocketChannel-class.html), and [Flutter AppLifecycleState API](https://api.flutter.dev/flutter/dart-ui/AppLifecycleState.html).
