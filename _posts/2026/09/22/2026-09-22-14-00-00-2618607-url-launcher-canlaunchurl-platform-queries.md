---
layout: post
title: "url_launcher canLaunchUrl False - Fix Flutter iOS and Android Release Links"
description: "Fix Flutter url_launcher links that work in debug but report canLaunchUrl false in production by configuring iOS schemes, Android queries, and safe fallback handling."
date: 2026-09-22
tags: [url_launcher, networking, Android, iOS, Flutter]
comments: true
share: true
---
![Flutter url_launcher platform configuration flow](https://images.unsplash.com/photo-1516321318423-f06f85e504b3?w=1200&q=80)

Flutter apps that open `mailto:`, `tel:`, or `sms:` links should not disable the button just because `canLaunchUrl` returns `false`. For an app that can provide a web fallback, calling `launchUrl` and handling its result is the safer default. If the UI must display whether a native app exists, add the iOS `LSApplicationQueriesSchemes` and Android `<queries>` declarations; otherwise the same code can look healthy in debug and fail after release. This does not apply to ordinary `https` links where a browser fallback is already enough.

## Why the result changes by platform

`canLaunchUrl` is a capability query, not a universal test that predicts every successful launch. On iOS, schemes passed to the query must be declared in `Info.plist`. On Android 11 (API 30) and newer, package visibility requires matching intent queries in `AndroidManifest.xml`. Missing either declaration can make the query return `false` even when a handler exists.

| Situation | Correct choice | Production consequence |
| --- | --- | --- |
| Open a web page | Call `launchUrl` directly | Keep the button enabled and show an error or fallback if it fails |
| Show “phone app installed” UI | Configure queries, then call `canLaunchUrl` | The query becomes meaningful for the declared scheme |
| Email from a support screen | Try `mailto`, then open an HTTPS form | Users without a mail app still have a path forward |
| Payment or custom app scheme | Declare only the required scheme and test a real device | Simulators may not have the target app installed |

## Configure the native capability queries

Add only the schemes that the app actually checks. This is a query configuration, not a declaration that your app handles those URLs.

For iOS, edit `ios/Runner/Info.plist`:

```xml
<key>LSApplicationQueriesSchemes</key>
<array>
    <string>mailto</string>
    <string>tel</string>
    <string>sms</string>
</array>
```

For Android, put `<queries>` directly under `<manifest>` in `android/app/src/main/AndroidManifest.xml`, not inside `<application>`:

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <queries>
        <intent>
            <action android:name="android.intent.action.VIEW" />
            <data android:scheme="mailto" />
        </intent>
        <intent>
            <action android:name="android.intent.action.VIEW" />
            <data android:scheme="tel" />
        </intent>
    </queries>

    <application
        android:label="example"
        android:name="${applicationName}"
        android:icon="@mipmap/ic_launcher">
        <!-- existing configuration -->
    </application>
</manifest>
```

The Android declaration matters for the query result. It is not a guarantee that a particular device has a mail or phone application installed.

## Prefer launch-then-fallback for user actions

The following helper tries the native email composer and falls back to a web form. The query is intentionally absent from the main action, so a missing declaration does not block a valid launch path.

```dart
import 'package:flutter/foundation.dart';
import 'package:url_launcher/url_launcher.dart';

Future<bool> openSupport() async {
  final email = Uri(
    scheme: 'mailto',
    path: 'support@example.com',
    query: _encodeQuery(<String, String>{
      'subject': 'Support request',
      'body': 'Describe the problem here',
    }),
  );

  if (await launchUrl(email)) {
    return true;
  }

  final webForm = Uri.parse('https://example.com/support');
  if (await launchUrl(webForm, webOnlyWindowName: '_blank')) {
    return true;
  }

  debugPrint('No support link could be opened');
  return false;
}

String _encodeQuery(Map<String, String> values) => values.entries
    .map((entry) =>
        '${Uri.encodeComponent(entry.key)}=${Uri.encodeComponent(entry.value)}')
    .join('&');
```

For non-HTTP schemes, the package documentation recommends encoding query values explicitly. That avoids spaces becoming `+` in places where the receiving app expects percent encoding. Replace the example address and form URL with your own endpoints; the fallback is a decision in your product flow, not a package feature.

## When `canLaunchUrl` is still useful

Use it when the result changes the UI, such as showing a “Call” row only when a phone handler is available. Configure every scheme used by that query, and treat `false` as “not confirmed,” not as proof that the later launch must fail. Do not use it as a required gate for a button that already has a fallback.

Before shipping, check the path that matches the feature:

- [ ] The scheme in the Dart `Uri` matches the iOS plist and Android manifest entries.
- [ ] Android `<queries>` is a direct child of `<manifest>`.
- [ ] Mail, phone, and SMS tests run on a physical device with the relevant app installed and absent.
- [ ] The fallback is tested after forcing `launchUrl` to return `false`.
- [ ] The release flavor uses the same native configuration as the debug flavor.

The key fix is separating “can I query this capability?” from “can I offer the user a working action?” Configure queries for the first question, and use direct launch plus an explicit fallback for the second.

Sources: [url_launcher configuration and API guidance](https://github.com/flutter/packages/blob/main/packages/url_launcher/url_launcher/README.md), [Apple `LSApplicationQueriesSchemes` reference](https://developer.apple.com/library/archive/documentation/General/Reference/InfoPlistKeyReference/Articles/LaunchServicesKeys.html).
