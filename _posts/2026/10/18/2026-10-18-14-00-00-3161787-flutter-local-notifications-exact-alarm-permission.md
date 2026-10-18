---
layout: post
title: "flutter_local_notifications Android 14 Exact Alarms - Keep Scheduled Reminders from Disappearing"
description: "Fix missing Flutter scheduled notifications on Android 13 and 14 by separating POST_NOTIFICATIONS from exact-alarm access and choosing a safe fallback."
date: 2026-10-18
tags: [flutter_local_notifications, Android, notifications, migration, debugging]
comments: true
share: true
---

![flutter_local_notifications Android exact alarm permission flow](https://images.unsplash.com/photo-1512941937669-90a1b58e7e9c?auto=format&fit=crop&w=1600&q=80)

*The important boundary is whether the reminder needs an exact delivery time or only an approximate one.*

If a Flutter reminder disappears on a fresh Android 13 or 14 install, adding another `show()` call is usually the wrong fix. `flutter_local_notifications` has two separate gates: `POST_NOTIFICATIONS` controls whether the user can see notifications, while exact-alarm access controls whether an exact scheduled alarm may be created. A calendar or medication app may need exact delivery; a news digest usually does not and should use an inexact schedule instead.

| Requirement | Android permission or mode | Good default |
|---|---|---|
| Display any notification | `POST_NOTIFICATIONS` | Request after explaining the value |
| Reminder at a precise time | `SCHEDULE_EXACT_ALARM` | Check and request only when needed |
| Approved alarm-clock style use case | `USE_EXACT_ALARM` | Use only if the app qualifies for store policy |
| “Around this time” reminder | `inexact` or `inexactAllowWhileIdle` | Avoid exact-alarm access |

## The failure that a debug install hides

Android 13 introduced runtime notification permission. Android’s exact-alarm behavior also changed: `SCHEDULE_EXACT_ALARM` is not pre-granted for most fresh installs targeting API 33 or higher. A schedule can therefore fail in two different ways: the alarm is created but hidden because notifications are denied, or the exact schedule is rejected before it is created.

Declare only the capabilities the app actually uses. For an app with precise reminders, the manifest can contain:

```xml
<manifest ...>
    <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
    <uses-permission android:name="android.permission.SCHEDULE_EXACT_ALARM" />
    <uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED" />

    <application ...>
        <receiver
            android:exported="false"
            android:name="com.dexterous.flutterlocalnotifications.ScheduledNotificationReceiver" />
        <receiver
            android:exported="false"
            android:name="com.dexterous.flutterlocalnotifications.ScheduledNotificationBootReceiver">
            <intent-filter>
                <action android:name="android.intent.action.BOOT_COMPLETED" />
                <action android:name="android.intent.action.MY_PACKAGE_REPLACED" />
            </intent-filter>
        </receiver>
    </application>
</manifest>
```

The receiver setup matters after a reboot or app replacement. It does not grant permission by itself.

## Ask for the two permissions independently

Request notification permission at a user-understandable point, then check exact-alarm access immediately before scheduling. The package exposes platform-specific methods for both checks:

```dart
final plugin = FlutterLocalNotificationsPlugin();

final android = plugin.resolvePlatformSpecificImplementation<
    AndroidFlutterLocalNotificationsPlugin>();

await android?.requestNotificationsPermission();

final exactGranted = await android?.canScheduleExactNotifications() ?? false;
if (!exactGranted) {
  await android?.requestExactAlarmsPermission();
}
```

Do not assume that a previous result remains valid. Android can revoke `SCHEDULE_EXACT_ALARM`, and backup/restore to a newer device does not preserve that grant in every case. Re-check it when the user enables a precise reminder and when the app resumes from settings.

## Choose a fallback instead of losing the reminder

The schedule mode should reflect the product promise. If the exact permission is unavailable, use an inexact mode only when the UI says “around” rather than “at exactly 9:00”. If exact timing is a hard requirement, keep the reminder pending and show a settings action instead of silently downgrading it.

```dart
final mode = exactGranted
    ? AndroidScheduleMode.exactAllowWhileIdle
    : AndroidScheduleMode.inexactAllowWhileIdle;

await plugin.zonedSchedule(
  reminder.id,
  reminder.title,
  reminder.body,
  reminder.time,
  const NotificationDetails(
    android: AndroidNotificationDetails(
      'reminders',
      'Reminders',
      channelDescription: 'Scheduled reminders',
    ),
  ),
  androidScheduleMode: mode,
  matchDateTimeComponents: DateTimeComponents.time,
);
```

This fallback is not equivalent. Idle mode and delivery precision are platform decisions, so record the selected mode in logs and test both branches. Also remember that an Android notification channel keeps its sound and vibration configuration after creation; changing those fields later with the same channel ID does not recreate the channel.

## Release checklist

- Test a clean install on Android 13+ with notification permission denied and allowed.
- Test exact access denied, granted, and revoked from system settings.
- Reboot the device and verify that a future schedule survives.
- Upgrade the app while a notification is scheduled.
- Confirm whether the product promise permits `inexact`; if not, block scheduling with a clear action.
- Test a channel created by an older release, because its sound settings are persistent.

The practical rule is simple: notification visibility, exact timing, and reboot recovery are three separate concerns. `flutter_local_notifications` provides the APIs, but the app must choose the permission path and the acceptable fallback explicitly.

Sources checked 2026-10-18: [flutter_local_notifications package documentation](https://pub.dev/packages/flutter_local_notifications), [Android exact alarm guidance](https://developer.android.com/develop/background-work/services/alarms), and [Android exact-alarm behavior changes](https://developer.android.com/about/versions/14/changes/schedule-exact-alarms).
