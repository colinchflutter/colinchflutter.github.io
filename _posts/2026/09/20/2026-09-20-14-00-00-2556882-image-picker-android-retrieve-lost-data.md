---
layout: post
title: "image_picker Android Process Death - Recover Lost Flutter Files with retrieveLostData"
description: "Fix Flutter image_picker uploads that lose their result after Android kills MainActivity by restoring LostDataResponse at startup and separating cancellation from recovery errors."
date: 2026-09-20
tags: [image_picker, Android, file_upload, debugging, recovery]
comments: true
share: true
---

![image_picker Android process death and lost-data recovery flow](https://pub.dev/static/img/pub-dev-icon.svg)

The production fix for a Flutter `image_picker` result that disappears after Android kills `MainActivity` is to call `retrieveLostData()` during Android startup, before the upload form is treated as empty. This applies to apps that pick an image or video through an external Android activity; it does not replace durable upload state, and it is not needed on iOS or Web.

The picture shows the boundary that matters: the picker can finish successfully while the original Flutter activity is gone. The selected file is then returned to a newly started activity through the plugin's recovery store, not through the `Future` that originally called `pickImage()`.

## Why a normal `await pickImage()` is not enough

`image_picker` uses Android intents such as `ACTION_GET_CONTENT` and `MediaStore.ACTION_IMAGE_CAPTURE`. While that external activity is open, Android can reclaim the app's `MainActivity` under memory pressure. When the picker finishes, Android starts the app again, but the Dart call stack that was awaiting the result no longer exists.

| Situation | What the original call sees | Correct handling |
| --- | --- | --- |
| User presses Back in the picker | `null` file | Treat as cancellation |
| Android recreates `MainActivity` | No result from the old `Future` | Call `retrieveLostData()` on startup |
| Recovery has a plugin/platform error | No usable file | Show retry state and log the exception code |
| File was recovered | `XFile` or a file list | Reattach it to the form and continue upload |

`retrieveLostData()` is Android-only. Calling it unconditionally makes a shared app fail on platforms where the method is not implemented, so the platform check belongs around the recovery call rather than inside the upload widget's error handling.

## Put recovery at the application boundary

Use one `ImagePicker` instance for the screen and run the recovery check when the screen is created. The recovery code must not assume that the user is still on the route that started the picker; Android recreated the activity, so route-local variables may have their initial values again.

This example supports both a single recovered file and multiple recovered files, and keeps a cancellation separate from an actual recovery failure.

```dart
import 'package:flutter/foundation.dart';
import 'package:flutter/material.dart';
import 'package:image_picker/image_picker.dart';

class AttachmentPage extends StatefulWidget {
  const AttachmentPage({super.key});

  @override
  State<AttachmentPage> createState() => _AttachmentPageState();
}

class _AttachmentPageState extends State<AttachmentPage> {
  final ImagePicker _picker = ImagePicker();
  List<XFile> _files = <XFile>[];
  String? _recoveryError;
  bool _recovering = true;

  @override
  void initState() {
    super.initState();
    _recoverLostPickerData();
  }

  Future<void> _recoverLostPickerData() async {
    if (!mounted || !defaultTargetPlatformIsAndroid) {
      if (mounted) setState(() => _recovering = false);
      return;
    }

    try {
      final LostDataResponse response = await _picker.retrieveLostData();
      if (response.isEmpty) return;

      final List<XFile> recovered = response.files ??
          (response.file == null ? <XFile>[] : <XFile>[response.file!]);

      if (response.exception != null) {
        setState(() => _recoveryError = response.exception!.code);
        return;
      }
      if (recovered.isNotEmpty && mounted) {
        setState(() => _files = recovered);
      }
    } on UnimplementedError {
      // The platform guard should prevent this; keep startup safe if it does not.
    } finally {
      if (mounted) setState(() => _recovering = false);
    }
  }

  Future<void> _pick() async {
    final XFile? file = await _picker.pickImage(source: ImageSource.gallery);
    if (file != null && mounted) {
      setState(() => _files = <XFile>[file]);
    }
  }

  @override
  Widget build(BuildContext context) {
    if (_recovering) return const Center(child: CircularProgressIndicator());
    return Column(
      children: <Widget>[
        if (_recoveryError != null) Text('Recovery failed: $_recoveryError'),
        Text('${_files.length} attachment(s) ready'),
        FilledButton(onPressed: _pick, child: const Text('Choose image')),
      ],
    );
  }
}

bool get defaultTargetPlatformIsAndroid =>
    !kIsWeb && defaultTargetPlatform == TargetPlatform.android;
```

The `finally` block matters. A recovery failure should not leave the page behind an indefinite loading indicator, and a recovered file should not be uploaded until the UI has received it. In a real form, replace `_files` with a repository or draft controller if the attachment must survive another activity recreation.

## The two traps that cause false fixes

### 1. Calling recovery after the first user action

Recovery is a startup concern. If the user reaches the page, taps “Choose image,” and only then does the app call `retrieveLostData()`, the old result may already have been ignored or overwritten by a new picker request. Run it once from the recreated app flow, then let the normal picker method handle new selections.

### 2. Saving the temporary path as permanent data

The package documentation describes the returned `XFile` as a single-session object. Copy the bytes to your own durable location or upload them promptly; do not store only the temporary path in a database and expect it to work after a later app launch. A recovered picker result proves that the selection was returned, not that your upload completed.

```dart
Future<void> uploadAttachment(XFile file) async {
  final bytes = await file.readAsBytes();
  // Send bytes or copy the file into app-owned storage here.
  // Persist an upload id/status separately from the picker result.
  debugPrint('Preparing ${file.name}: ${bytes.length} bytes');
}
```

For a multi-file picker, treat `response.files` as the authoritative recovered list. The Android implementation can only recover what the plugin retained; do not silently invent missing entries or mark the entire draft as uploaded.

## A reproducible release check

Hot reload and a routine background/foreground cycle do not prove this path. The failure requires the external picker plus an activity/process recreation. Keep this checklist beside the Android release test matrix:

1. Start a release or profile build on a memory-constrained Android device or emulator.
2. Open the gallery from the attachment page and select a large image.
3. While the external picker is open, stop the app process using Android device tools, then finish the picker flow if the device permits it.
4. Relaunch the app and verify that the recovered file appears before the upload form is submitted.
5. Repeat with Back/cancel and with a plugin exception. Those paths must not create a phantom attachment.

The official package guidance says Android may kill `MainActivity` under high memory pressure and recommends `ImagePicker.retrieveLostData()`. The API is a recovery handoff, not a general crash-recovery system: persist the draft, upload status, and server-side id separately when the workflow matters.

### Short decision checklist

- Android and external picker: call `retrieveLostData()` during startup.
- iOS, Web, or desktop: skip the Android-only API.
- `response.isEmpty`: no pending selection; keep the form empty.
- `response.exception != null`: show retry/error state, not success.
- `response.file` or `response.files`: restore the form, then copy/upload promptly.
- Need recovery after another restart: store draft metadata and upload status outside `image_picker`.

Sources: [image_picker package documentation](https://pub.dev/packages/image_picker), [Flutter image_picker API source](https://github.com/flutter/packages/blob/main/packages/image_picker/image_picker/lib/image_picker.dart), and [ImagePickerAndroid API reference](https://pub.dev/documentation/image_picker_android/latest/image_picker_android/ImagePickerAndroid-class.html) (checked 2026-09-20).
