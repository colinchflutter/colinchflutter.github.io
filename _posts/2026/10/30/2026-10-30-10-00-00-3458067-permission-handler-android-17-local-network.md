---
layout: post
title: "permission_handler Android 17 Local Network - Fix Flutter Socket Failures Before They Ship"
description: "Migrate Flutter LAN device discovery to permission_handler 13, declare ACCESS_LOCAL_NETWORK, and request it before Dart sockets connect on Android 17."
date: 2026-10-30
tags: [permission_handler, Android, networking, migration, Flutter]
comments: true
share: true
---

![permission_handler Android 17 local network permission flow for Flutter](https://images.unsplash.com/photo-1516321318423-f06f85e504b3?auto=format&fit=crop&w=1600&q=80)

If a Flutter app discovers or connects to printers, NAS devices, cameras, or other LAN endpoints, upgrade to `permission_handler` 13 and request `ACCESS_LOCAL_NETWORK` before opening a Dart socket when targeting Android 17 (API 37) or newer. This does not apply to ordinary HTTPS calls to a remote server, and it does not replace the device protocol's own authentication.

## The failure boundary

Starting with Android 17, local network access is blocked by default for apps targeting API 37+. The important trap is that `dart:io` sockets cannot display an Android permission prompt. If the socket is created first, the connection can fail with `SocketException`; showing a retry button after that is too late for a clean first-run flow.

| App behavior | Required action |
| --- | --- |
| HTTPS API on the public internet | No local-network permission |
| Connect to `192.168.x.x` or a `.local` device | Declare and request local-network access |
| Target below Android 17 | Keep the existing path, but test the upgrade boundary |
| Target Android 17+ with no LAN feature | Do not add the permission just because the package supports it |

## Migration steps

The manifest declaration belongs inside the `<manifest>` element. The package changelog records both the Android 17 permission support and the compile SDK 37 requirement in the 13.0.0 release.

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <uses-permission android:name="android.permission.ACCESS_LOCAL_NETWORK" />
    <application
        android:label="my_app"
        android:name="${applicationName}"
        android:icon="@mipmap/ic_launcher">
        <!-- existing configuration -->
    </application>
</manifest>
```

Request the permission immediately before the first LAN connection, not during every widget rebuild. The request result is the decision that matters; a previous `check` alone is not enough for the initial prompt.

```dart
import 'dart:io';
import 'package:permission_handler/permission_handler.dart';

Future<Socket> connectToDevice(String host, int port) async {
  if (Platform.isAndroid) {
    final status = await Permission.accessLocalNetwork.request();
    if (!status.isGranted) {
      throw StateError('Local network access was not granted');
    }
  }

  return Socket.connect(host, port, timeout: const Duration(seconds: 5));
}
```

Treat denial as a product state. Explain why the app needs to find a local device, offer a settings path only when the status indicates that the user must change it there, and keep remote-only features usable. Do not silently retry the socket: repeated failures can look like a device or firewall problem when the missing input is an Android permission.

## Reproducible release check

- Set the Android compile/target SDK to the level required by the package and your release toolchain.
- Test a fresh install targeting API 37+ with the permission absent, then with it denied.
- Test the exact LAN workflow on a real device; an emulator route can hide discovery and multicast differences.
- Verify that a remote HTTPS request still works when local-network access is denied.
- Record the host, port, permission result, and socket exception without logging credentials or device payloads.

The key distinction is scope: `permission_handler` supplies the runtime permission bridge, while the app still owns discovery, timeout, protocol authentication, and the user-facing fallback. The official Flutter guidance and the package changelog were checked on 2026-10-30.

Sources: [Flutter local network permission guide](https://docs.flutter.dev/platform-integration/android/local-network-permission), [`permission_handler` changelog](https://pub.dev/packages/permission_handler/changelog), and [`permission_handler` package documentation](https://pub.dev/packages/permission_handler).
