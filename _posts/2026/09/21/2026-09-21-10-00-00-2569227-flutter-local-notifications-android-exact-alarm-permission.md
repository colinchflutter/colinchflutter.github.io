---
layout: post
title: "flutter_local_notifications Android 14 - Fix Scheduled Notifications with the Right Alarm Permission"
description: "Fix flutter_local_notifications scheduled notifications that stop on Android 13 or 14 by separating POST_NOTIFICATIONS, exact-alarm access, manifest receivers, and the inexact fallback."
date: 2026-09-21
tags: [flutter_local_notifications, Android, notifications, migration, debugging]
comments: true
share: true
---

![flutter_local_notifications Android scheduled notification permission flow](https://pub.dev/static/img/pub-dev-icon.svg)

If a Flutter app schedules reminders for a user-selected minute, keep `flutter_local_notifications` with `SCHEDULE_EXACT_ALARM` and handle denial. If a reminder can arrive within a time window, choose an inexact schedule instead and avoid asking for special access. `POST_NOTIFICATIONS` alone does not grant exact-alarm access, and adding both permissions to every app is not a safe fix.

## The failure that looks like a plugin bug

The symptom is usually “`show()` works, but `zonedSchedule()` does nothing after upgrading a phone or changing the target SDK.” There are four independent gates:

| Gate | What it controls | Typical failure |
|---|---|---|
| `POST_NOTIFICATIONS` | Whether Android may show notifications | Scheduled work exists but is invisible |
| `SCHEDULE_EXACT_ALARM` | Whether exact alarms may be created | A platform exception or skipped schedule |
| Scheduled receivers | Delivery and reboot rescheduling | Works until the process or device restarts |
| Battery/OEM policy | Delivery timing while idle | Arrives late or in a batch |

Android 13 introduced notification runtime permission. Android 14 also stopped pre-granting `SCHEDULE_EXACT_ALARM` to most newly installed apps targeting Android 13 or higher. These are separate checks, so granting notification permission cannot repair an exact-alarm denial ([Android notification permission](https://developer.android.com/develop/ui/compose/notifications/notification-permission), [Android exact alarms](https://developer.android.com/about/versions/14/changes/schedule-exact-alarms)).

## Keep only the manifest entries the feature needs

Since `flutter_local_notifications` 16, feature-specific permissions and receivers belong in the app's `android/app/src/main/AndroidManifest.xml`. For scheduled notifications, the minimum shape is:

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
    <uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED" />
    <uses-permission android:name="android.permission.SCHEDULE_EXACT_ALARM" />

    <application>
        <receiver
            android:exported="false"
            android:name="com.dexterous.flutterlocalnotifications.ScheduledNotificationReceiver" />
        <receiver
            android:exported="false"
            android:name="com.dexterous.flutterlocalnotifications.ScheduledNotificationBootReceiver">
            <intent-filter>
                <action android:name="android.intent.action.BOOT_COMPLETED" />
                <action android:name="android.intent.action.MY_PACKAGE_REPLACED" />
                <action android:name="android.intent.action.QUICKBOOT_POWERON" />
                <action android:name="com.htc.intent.action.QUICKBOOT_POWERON" />
            </intent-filter>
        </receiver>
    </application>
</manifest>
```

The manifest is not enough. Request notification permission on Android 13+ at a user-relevant moment, then request exact-alarm access only when the product truly promises a precise time:

```dart
final android = plugin.resolvePlatformSpecificImplementation<
    AndroidFlutterLocalNotificationsPlugin>();
var exactGranted = false;

if (android != null) {
  await android.requestNotificationsPermission();

  // Use this only for an alarm-clock-like, user-visible exact reminder.
  exactGranted = await android.requestExactAlarmsPermission() ?? false;
  if (!exactGranted) {
    // Keep the reminder, but reschedule it as inexact or ask the user to retry.
  }
}
```

Do not declare `USE_EXACT_ALARM` just to remove the dialog. Android documents that permission for calendar and alarm-clock apps, and app-store policy can audit whether the use case qualifies. For ordinary reminders, `SCHEDULE_EXACT_ALARM` plus a fallback is easier to justify.

## Choose the schedule mode from the product requirement

`AndroidScheduleMode.exactAllowWhileIdle` is appropriate only when the time itself is part of the promise. For a “remind me around 9 AM” feature, use an inexact mode and communicate that Android may delay delivery while idle. This avoids a permission request, but it does not defeat OEM battery restrictions.

```dart
await plugin.zonedSchedule(
  42,
  'Daily check-in',
  'Open the app when convenient',
  scheduledDate,
  notificationDetails,
  androidScheduleMode: exactGranted
      ? AndroidScheduleMode.exactAllowWhileIdle
      : AndroidScheduleMode.inexactAllowWhileIdle,
  uiLocalNotificationDateInterpretation:
      UILocalNotificationDateInterpretation.absoluteTime,
);
```

The exact branch must not run before permission is known. Also re-check after the user returns from Settings: Android can revoke special access, and the plugin documents that an exact schedule may stop being scheduled when that access is revoked.

## Reproduce the upgrade failure without reinstalling everything

Use a debug package ID in place of `PACKAGE_NAME`. The first group resets the Android 13 notification prompt to the “new install” state:

```bash
adb shell pm revoke PACKAGE_NAME android.permission.POST_NOTIFICATIONS
adb shell pm clear-permission-flags PACKAGE_NAME \
  android.permission.POST_NOTIFICATIONS user-set
adb shell pm clear-permission-flags PACKAGE_NAME \
  android.permission.POST_NOTIFICATIONS user-fixed
```

On Android 14 or higher, open **Settings > Apps > Special app access > Alarms & reminders**, disable the app, and schedule the same reminder again. A useful test matrix is:

| Notification permission | Exact-alarm access | Expected result |
|---|---|---|
| Denied | Granted | Alarm may be scheduled, but the notification is hidden |
| Granted | Denied | Inexact fallback can work; exact scheduling must not be attempted |
| Granted | Granted | Exact scheduling is eligible, subject to device idle policy |
| Granted | Revoked after scheduling | Re-check on resume and recreate or downgrade the reminder |

This matrix prevents a misleading “it worked once” result. An existing install may retain a permission after an OS upgrade, while a fresh install on Android 14 starts with exact-alarm access denied for most apps targeting Android 13 or higher. Android’s own testing guidance recommends disabling the special access and observing the app’s behavior.

## Migration choices that should be explicit

| Product promise | Permission choice | Flutter behavior |
|---|---|---|
| Medication or alarm-clock time is exact | `SCHEDULE_EXACT_ALARM` | Request, verify, then use an exact schedule |
| Calendar/alarm-clock app qualifies for policy | `USE_EXACT_ALARM` | No user prompt, but store eligibility must be reviewed |
| Reminder may drift by a few minutes | No exact permission | Use an inexact schedule and explain the timing |
| Background sync, not user-visible alert | No local alarm | Use a background-work API with its own limits |

The last row matters because a local notification is not a general-purpose background execution guarantee. If the job is uploading logs or refreshing data, an exact alarm is the wrong recovery path even if it makes a demo look reliable.

## A release-build checklist that catches the real regressions

- Test a fresh install on Android 13+ with notification permission denied.
- Test Android 14+ with **Alarms & reminders** disabled, then grant it and retry.
- Kill the app, schedule a reminder, and reboot the device.
- Confirm the notification channel is created with the intended importance.
- Build the release variant and keep the notification icon resources from R8 removal.
- Test the fallback path rather than treating permission denial as an exception-free success.

The package documentation and changelog are the source of truth for the current receiver names and renamed `requestNotificationsPermission()` API ([package setup](https://pub.dev/packages/flutter_local_notifications), [changelog](https://pub.dev/packages/flutter_local_notifications/changelog)). The version matters: older snippets may assume the plugin supplied every receiver and permission automatically.

The short rule is simple: notification visibility, exact scheduling, reboot recovery, and battery behavior are different contracts. Fix the contract that failed instead of adding every permission to the manifest.
