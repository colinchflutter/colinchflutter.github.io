---
layout: post
title: "record Flutter Android 14 Background Recording - Fix Microphone Foreground Service Starts"
description: "Use Flutter record safely on Android 14 by separating foreground recording from background capture, checking microphone permission before start, and testing the foreground-service boundary."
date: 2026-11-04
tags: [record, Android, migration, performance, testing]
comments: true
share: true
---

![Flutter record package handling Android microphone recording](https://images.unsplash.com/photo-1590602847861-f357a9332bbc?auto=format&fit=crop&w=1600&q=80)

*The failure boundary is the moment a recorder leaves the visible Flutter activity and needs Android foreground-service privileges.*

If `record` works while the Flutter screen is visible but fails when the user locks the phone, the fix is not a longer timeout. Choose the product behavior first: foreground-only recording can use `AudioRecorder` directly; recording that must continue after the screen is hidden needs a microphone foreground service, the right manifest declarations, and a start initiated while the app is visible. Android 14 can throw `SecurityException` before the recorder reaches its first audio frame.

This applies to voice notes, interviews, and continuous capture. It does not apply to a short recording that must stop when the user leaves the page. Adding a foreground service to every recorder increases notification and lifecycle complexity.

## The two recording contracts

The `record` package exposes a clear Dart API, but “record in the background” is a platform contract rather than a `RecordConfig` flag. Keep these cases separate:

| Product requirement | Package/platform path | Failure to expect |
| --- | --- | --- |
| Record while the page is visible | `AudioRecorder.start()` after `hasPermission()` | Missing runtime microphone permission |
| Continue with screen locked | Android microphone foreground service plus `record` | Service start rejected or notification missing |
| Start from a background callback | Start only through an Android-allowed user/system path | Android 12+ background-start restriction |
| Record after the app is killed | Persist a draft and define a restart policy | No Dart object can survive process death |

The package documentation lists Android support and requires permission checking before `start()`. Flutter’s official recipe also treats `AudioRecorder` as the owner of a normal recording session. That is enough for an in-app voice note, not proof that arbitrary background capture is available.

## Keep the visible-session path deterministic

Request permission before constructing a recording state that the UI treats as active. Also dispose the recorder when the feature is finished; otherwise an old native recorder can keep the microphone busy for the next screen.

```dart
final recorder = AudioRecorder();

Future<bool> startVisibleRecording(String path) async {
  if (!await recorder.hasPermission()) {
    return false;
  }

  await recorder.start(
    const RecordConfig(
      encoder: AudioEncoder.aacLc,
      numChannels: 1,
      sampleRate: 44_100,
    ),
    path: path,
  );
  return true;
}

Future<void> closeRecorder() async {
  await recorder.stop();
  await recorder.dispose();
}
```

The `true` return here means only that the native start call completed. It does not mean the user granted a background capability, the file is durable, or a recording notification is visible. Keep those states separate in the app model.

## Android 14 changes the order of operations

For a microphone foreground service, Android 14 checks both the service type and the required permission. Android’s current guidance says a service using the microphone must declare the microphone type and its corresponding permission. It also says a while-in-use microphone permission is not usable for creating the service from an ordinary background state.

The minimum manifest shape for an app that owns such a service is conceptually:

```xml
<uses-permission android:name="android.permission.RECORD_AUDIO" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_MICROPHONE" />

<service
    android:name=".RecordingService"
    android:foregroundServiceType="microphone"
    android:exported="false" />
```

The exact service class belongs to the background-recording integration you choose; do not paste this block if the plugin already declares a service. Duplicate declarations are less useful than inspecting the merged manifest and the actual service type. The important sequence is:

1. Bring the activity to the foreground.
2. Request and confirm `RECORD_AUDIO`.
3. Start the microphone foreground service.
4. Only then attach the native recorder and expose “recording” in the UI.

Starting from a timer, an isolate, or a silent background callback can fail even when a permission check reports granted. Android 14 evaluates the app’s current visibility when the foreground service is created.

## Make failure and recovery visible

Do not keep the UI in `recording` after `start()` throws. Save a pending draft state before the service boundary, then reconcile it with the actual native state when the app returns.

```dart
Future<void> beginRecording() async {
  await draftStore.markPending();

  try {
    if (!await recorder.hasPermission()) {
      await draftStore.markRejected('microphone permission');
      return;
    }

    // If background capture is required, start the Android service here
    // while the activity is visible, before attaching the recorder.
    await recorder.start(const RecordConfig(), path: await draftStore.path());
    await draftStore.markActive();
  } catch (error, stackTrace) {
    await draftStore.markFailed(error.toString());
    reportRecordingFailure(error, stackTrace);
  }
}
```

For a long recording, write to app-owned storage rather than a cache directory. On `stop()`, verify that the returned path exists and has non-zero length before marking the upload ready. If the process dies, mark the draft as interrupted and offer recovery; do not claim that a partial container is playable.

## Reproduction matrix

| Test | Expected evidence |
| --- | --- |
| Android 13, visible activity | Permission prompt and file creation |
| Android 14, visible activity | Service starts with microphone type |
| Android 14, start from background | Explicit rejection or an allowed user-triggered path |
| Lock screen after start | Recording continues and notification remains visible |
| Deny microphone permission | No native recorder is attached; draft is rejected |
| Stop, relaunch, upload | File exists in app-owned storage and is validated before upload |

The practical fix is to treat `record` as the audio engine, not as a bypass around Android lifecycle rules. Foreground-only notes need permission, a deterministic start, and disposal. Background capture needs a visible-started microphone foreground service, Android 14 declarations, and a recovery state for process death.

References checked 2026-11-04: [`record` package documentation](https://pub.dev/packages/record), [Flutter’s record recipe](https://docs.flutter.dev/cookbook/audio/record), [Android foreground-service type changes](https://developer.android.com/develop/background-work/services/fgs/changes), and [Android background-start restrictions](https://developer.android.com/develop/background-work/services/fgs/restrictions-bg-start).
