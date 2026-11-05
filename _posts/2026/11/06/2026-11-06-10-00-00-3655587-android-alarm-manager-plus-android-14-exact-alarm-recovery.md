---
layout: post
title: "android_alarm_manager_plus Android 14 - Recover Flutter Alarms After Exact-Alarm Denial"
description: "Fix android_alarm_manager_plus alarms that stop on Android 14 by separating exact-alarm permission, release callbacks, and the inexact fallback path."
date: 2026-11-06
tags: [android_alarm_manager_plus, Android, background, migration, testing, Flutter]
comments: true
share: true
---

![android_alarm_manager_plus exact alarm recovery flow on Android](https://images.unsplash.com/photo-1512941937669-90a1b58e7e9c?auto=format&fit=crop&w=1600&q=80)

If a Flutter app uses `android_alarm_manager_plus` for a user-visible alarm that must fire at a precise time, handle Android 14's exact-alarm access before scheduling. If the task is only a sync, log upload, or approximate reminder, keep the permission out of the app and use an inexact alarm or WorkManager instead. The plugin starts the alarm service, but it does not request `SCHEDULE_EXACT_ALARM` for you.

## The failure is a policy change, not a Dart timer bug

The current `android_alarm_manager_plus` documentation says that Android 14 denies `SCHEDULE_EXACT_ALARM` by default for most fresh installs targeting Android 13 or newer. Android's own guidance says the permission can also be denied after backup and restore, and that exact scheduling must be checked before calling the exact API.

That creates three separate failure points:

| Symptom | Actual boundary | Correct response |
| --- | --- | --- |
| `oneShot` or `periodic` fails on a new Android 14 install | Special app access is denied | Explain the need and open the Alarms & reminders settings |
| Alarm works in debug but not in release | Callback was removed by tree shaking | Add `@pragma('vm:entry-point')` to the top-level callback |
| Callback runs but cannot read app state | Alarm runs in another isolate | Initialize the required plugin and pass durable state explicitly |

Do not “fix” the first row by adding `USE_EXACT_ALARM` to every app. Android reserves that permission for qualifying alarm-clock and calendar use cases, and store policy can matter. A normal reminder app should usually request `SCHEDULE_EXACT_ALARM`, or avoid exact scheduling when its product promise allows a time window.

## Keep initialization and the callback boring

The callback below is deliberately top-level. It does not capture a widget, a provider container, or a singleton created by `main()`. The alarm callback is a separate Dart isolate, so those objects are not automatically available.

```dart
import 'dart:developer' as developer;

import 'package:android_alarm_manager_plus/android_alarm_manager_plus.dart';
import 'package:flutter/widgets.dart';

@pragma('vm:entry-point')
void reminderAlarmCallback() {
  developer.log('Reminder alarm fired', name: 'reminder_alarm');

  // Read durable input here, or send a message to the UI isolate.
  // Do not assume that a repository created in main() exists in this isolate.
}

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await AndroidAlarmManager.initialize();
  runApp(const App());
}

Future<bool> scheduleReminder(DateTime time) {
  final delay = time.difference(DateTime.now());
  if (delay.isNegative) {
    return Future.value(false);
  }

  return AndroidAlarmManager.oneShot(
    delay,
    1001,
    reminderAlarmCallback,
    exact: true,
    allowWhileIdle: true,
    wakeup: true,
  );
}
```

`exact: true` is a product decision. `allowWhileIdle: true` is useful when the alarm should still be considered during Doze, but it does not remove the permission requirement or guarantee unlimited alarm frequency. Also keep the ID stable: scheduling another alarm with the same ID replaces the previous one, so generate IDs from the user's reminder identity rather than from a rebuild.

## Check access before calling the plugin

The plugin does not expose the Android 14 special-access flow as a complete permission manager. Put that decision in a small Android policy class or use a permission package that supports `scheduleExactAlarm`. The important part is the order: check, request, re-check after the app resumes, then schedule.

```dart
import 'package:permission_handler/permission_handler.dart';

Future<bool> ensureExactAlarmAccess() async {
  final permission = Permission.scheduleExactAlarm;
  if (await permission.isGranted) {
    return true;
  }

  final result = await permission.request();
  return result.isGranted;
}

Future<bool> createPreciseReminder(DateTime time) async {
  if (!await ensureExactAlarmAccess()) {
    return false; // The UI can offer an inexact reminder instead.
  }

  return scheduleReminder(time);
}
```

Treat the returned `false` as a normal product state, not as an exception to hide. Save the reminder definition in durable storage, show that exact timing is disabled, and offer a fallback. If the user grants access in Android Settings, run the same check again from `AppLifecycleState.resumed` before rescheduling. Android can revoke this access, and existing alarms may be canceled when the permission is revoked.

For a fallback, remove `exact: true` and use an inexact alarm only when “around this time” is acceptable. For data synchronization or uploads, use WorkManager rather than an exact alarm. Android's guidance specifically points to inexact alarms and WorkManager for work that does not need a precise user-facing trigger.

## Reproduce the release-only failure before shipping

Use a real Android 14-or-newer device or emulator and test the permission boundary explicitly. The following checklist is small enough to keep in a release test plan:

```text
[ ] Fresh install with Alarms & reminders disabled
[ ] Tap “schedule precise reminder”
[ ] Confirm the app explains the setting before opening Android Settings
[ ] Grant access, return to the app, and verify the reminder is scheduled
[ ] Revoke access, resume the app, and verify the fallback appears
[ ] Run the same flow with a release APK
[ ] Force-stop/reopen the app and verify the saved reminder can be rebuilt
[ ] Test after backup/restore to an Android 14 device
```

On a test device, Android's documentation recommends disabling the app's Alarms & reminders access from Special app access. That is more representative than repeatedly pressing the schedule button in a debug session. If the release callback never logs, inspect the entry-point pragma first. If the callback logs but the reminder is stale, inspect the storage and isolate boundary rather than the alarm permission.

## The practical decision

`android_alarm_manager_plus` is a good fit when a Flutter callback must be triggered from Android's AlarmManager and the product can own Android-specific permission UX. It is not a general background execution guarantee, and exact alarms are expensive enough that Android recommends avoiding them when timing can be flexible.

| Requirement | Choice |
| --- | --- |
| Alarm-clock or calendar action at a precise time | `android_alarm_manager_plus` with checked `SCHEDULE_EXACT_ALARM` |
| Reminder can drift within a window | Inexact alarm without special access |
| Upload, cleanup, or periodic server sync | WorkManager |
| Release callback must survive tree shaking | Top-level function plus `@pragma('vm:entry-point')` |
| Callback needs app data | Durable storage or explicit isolate messaging |

The reliable fix is therefore a three-part boundary: request exact-alarm access only for a genuine precise-time feature, make the callback independently reachable in release builds, and keep a tested fallback for denial. Updating the package alone cannot supply those product and Android lifecycle decisions.

Sources checked 2026-11-06: [`android_alarm_manager_plus` documentation](https://pub.dev/packages/android_alarm_manager_plus), [Android exact-alarm behavior changes](https://developer.android.com/about/versions/14/changes/schedule-exact-alarms), and [Android alarm scheduling guidance](https://developer.android.com/develop/background-work/services/alarms).
