---
layout: post
title: "geolocator Background Location - Fix Android Permission Requests That Always Stay Denied"
description: "Use geolocator to request Android foreground and background location in stages, verify Android 10+ manifest rules, and avoid streams that fail after a target SDK migration."
date: 2026-10-04
tags: [geolocator, Android, location, permissions, migration, Flutter]
comments: true
share: true
---
![Flutter geolocator Android background location permission flow](https://images.unsplash.com/photo-1524666041070-9b8767495eab?auto=format&fit=crop&w=1600&q=80)

If a Flutter feature only needs the user’s position while a screen is open, request foreground location with `geolocator` and stop there. If it records a route, geofences a home, or shares movement while the screen is off, request foreground access first and background access as a separate product step. Android 11 and newer can ignore a request that asks for both at once, so adding every manifest permission and calling `requestPermission()` once is not a reliable fix.

The important boundary is not the Dart enum. It is the sequence:

```text
feature explanation → foreground permission → active use case → background permission → stream/service
```

This applies to Android apps using `geolocator` for background updates. It does not turn an ordinary foreground-only map into a background service, and it does not satisfy Google Play’s separate policy review for background location.

## What changes across Android versions

`geolocator`’s current documentation separates the manifest declarations from the runtime flow. `ACCESS_COARSE_LOCATION` is enough for approximate foreground access, while precise positioning normally adds `ACCESS_FINE_LOCATION`. Android 10 (API 29) introduced `ACCESS_BACKGROUND_LOCATION`, and Android 14 (API 34) adds foreground-service requirements for continuous location work.

| Product need | Manifest/runtime decision | Practical result |
| --- | --- | --- |
| Show nearby stores while the page is open | `COARSE` or `FINE`; request foreground | No background permission or service |
| Record a workout with the screen locked | Foreground permission, then background access when the user enables tracking | A visible tracking policy is required |
| Geofence a home or device | Foreground permission, then background access for the geofence feature | Do not request it during onboarding without context |
| Upload periodic analytics | Location permission alone is not a scheduler | Choose a background execution design separately |

Android’s own guidance says that requesting foreground and background location together on Android 11+ can result in neither permission being granted. That is the failure mode that often appears after a target SDK or plugin migration: the code still compiles, but the permission result never reaches the expected `always` state.

## Keep the manifest narrow

Add only the declarations required by the feature. `geolocator`’s plugin manifest can contribute some service-related entries, but the app still owns its location contract and its runtime request.

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
    <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />

    <!-- Only for a feature that genuinely works while the app is not in use. -->
    <uses-permission android:name="android.permission.ACCESS_BACKGROUND_LOCATION" />

    <application
        android:label="example"
        android:name="${applicationName}"
        android:icon="@mipmap/ic_launcher">
        <!-- Flutter activity configuration remains here. -->
    </application>
</manifest>
```

Do not add `ACCESS_BACKGROUND_LOCATION` just to make a denied foreground request disappear. It expands the privacy and store-review surface without changing the foreground request sequence.

## Model the request as two user actions

The first action should explain a visible feature, such as “show my current position on the map.” After that permission is granted, the user can enable a second feature such as “keep recording while the screen is locked.” That makes the later `always` request understandable and gives the app a clean place to handle refusal.

The following service keeps the platform checks and the staged requests in one place. It intentionally returns a decision instead of starting a stream as a side effect of a permission dialog.

```dart
import 'package:geolocator/geolocator.dart';

enum LocationAccess {
  unavailable,
  foreground,
  background,
}

class LocationAccessService {
  Future<LocationAccess> requestForeground() async {
    if (!await Geolocator.isLocationServiceEnabled()) {
      return LocationAccess.unavailable;
    }

    var permission = await Geolocator.checkPermission();
    if (permission == LocationPermission.denied) {
      permission = await Geolocator.requestPermission();
    }

    if (permission == LocationPermission.denied ||
        permission == LocationPermission.deniedForever) {
      return LocationAccess.unavailable;
    }

    return switch (permission) {
      LocationPermission.whileInUse || LocationPermission.always =>
        LocationAccess.foreground,
      LocationPermission.denied || LocationPermission.deniedForever =>
        LocationAccess.unavailable,
      LocationPermission.unableToDetermine => LocationAccess.unavailable,
    };
  }

  Future<LocationAccess> requestBackground() async {
    final foreground = await requestForeground();
    if (foreground != LocationAccess.foreground) {
      return foreground;
    }

    final permission = await Geolocator.checkPermission();
    if (permission == LocationPermission.always) {
      return LocationAccess.background;
    }

    // Android may open Settings instead of showing a second dialog.
    final upgraded = await Geolocator.requestPermission();
    return upgraded == LocationPermission.always
        ? LocationAccess.background
        : LocationAccess.foreground;
  }
}
```

The enum is a product boundary, not a guarantee that updates will continue forever. On Android, `whileInUse` can be the correct result for a map screen. A background tracker should not silently treat it as `always` and start claiming that it is recording.

## Start the stream only after access is known

Keep the permission decision separate from `getPositionStream`. This prevents a denied request from creating a subscription that looks healthy but never emits useful positions.

`AndroidSettings` and `ForegroundNotificationConfig` are platform-specific types. Add a compatible `geolocator_android` dependency when importing them directly; the package documentation treats that platform implementation as an explicit import boundary.

```dart
import 'package:geolocator/geolocator.dart';
import 'package:geolocator_android/geolocator_android.dart';

final access = await locationAccessService.requestBackground();

if (access != LocationAccess.background) {
  // Show the feature-specific explanation or an App Settings action.
  return;
}

final settings = const AndroidSettings(
  accuracy: LocationAccuracy.high,
  distanceFilter: 25,
  intervalDuration: Duration(seconds: 10),
  foregroundNotificationConfig: ForegroundNotificationConfig(
    notificationTitle: 'Route tracking is active',
    notificationText: 'Location is being used while the screen is off.',
    enableWakeLock: false,
  ),
);

final subscription = Geolocator.getPositionStream(
  locationSettings: settings,
).listen(
  savePosition,
  onError: handleLocationError,
);
```

A foreground service notification is part of the user-visible behavior of continuous Android tracking. It is not a replacement for `ACCESS_BACKGROUND_LOCATION`, and a location stream is not a general background job runner. If the real requirement is a once-per-day sync, a location stream is the wrong tool.

## Migration traps worth checking

| Symptom | Likely boundary | Fix |
| --- | --- | --- |
| Both permissions remain denied on Android 11+ | Foreground and background requested together | Split the request into two user actions |
| Approximate location arrives after requesting precise location | User selected approximate access on Android 12+ | Treat accuracy as a product state; request both manifest permissions but handle coarse results |
| The stream stops on Android 14 | Background/service configuration is incomplete | Confirm the foreground service type and test a locked-screen release build |
| `deniedForever` never shows a dialog | The user must change the setting outside the app | Offer `Geolocator.openAppSettings()` and explain why |
| Location permission exists but Play review fails | Background location is not core to the app | Remove the permission or prepare the required policy disclosure |

Do not use a successful `checkPermission()` call as proof that the service is permitted to run in every lifecycle state. Check the location service switch, current permission, stream errors, and the Android build configuration separately.

## A reproducible release checklist

- Test a fresh install on Android 11 or newer, not only an upgraded debug install.
- Grant foreground location, use the foreground feature, then enable background tracking.
- Repeat with approximate location on Android 12+.
- Deny the second request and verify that the app does not label the route as active.
- Lock the screen and verify the foreground notification and stream error path.
- Test a target SDK 34+ release build on a device or emulator that has never granted the permission.
- Revoke permission from system settings, relaunch, and verify the recovery UI.
- Check that the Play listing and in-app explanation justify background access if the feature requires it.

The official `geolocator` documentation and Android permission guidance were checked on 2026-10-04. The reusable rule is simple: ask for the smallest location capability that unlocks the current feature, then make the background upgrade an explicit action. That sequence survives target SDK changes better than a one-time request for every permission.

Sources: [`geolocator` package documentation](https://pub.dev/packages/geolocator), [Android runtime location permission guidance](https://developer.android.com/develop/sensors-and-location/location/permissions/runtime), and [Android foreground-service permission guidance](https://developer.android.com/develop/background-work/services/fgs/permissions).
