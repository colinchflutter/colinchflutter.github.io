---
layout: post
title: "Flutter camera Lifecycle - Rebuild CameraController Safely After Resume"
description: "Fix Flutter camera preview failures after backgrounding by disposing CameraController on inactive, reinitializing on resume, and guarding async races."
date: 2026-09-30
tags: [camera, Flutter, Android, iOS, performance, debugging]
comments: true
share: true
---

![Flutter camera lifecycle and controller reinitialization](https://images.unsplash.com/photo-1516035069371-29a1b244cc32?w=1200&q=80)

The `camera` package is a good fit for a live preview, but apps that must survive a phone call, app switch, or permission screen should own the controller lifecycle. Dispose the `CameraController` when the app becomes inactive, create a new one after `resumed`, and ignore an initialization that finishes after the page has already become inactive. This fix applies to a real camera preview; a one-shot `image_picker` flow does not need a long-lived controller.

## Why the preview breaks after returning

Since `camera` 0.5.0, lifecycle changes are not handled by the plugin. The official package page explicitly puts that responsibility on the application. A controller can therefore still hold a native camera resource while the OS has taken it away. The visible symptoms vary by platform:

| Situation | Unsafe assumption | Result | Correct action |
| --- | --- | --- | --- |
| App becomes inactive | The old controller will resume | Preview stays black or reports a camera error | Dispose it |
| App resumes | `initialize()` can be called on the same instance | Native session remains invalid | Create a new controller |
| Initialization completes late | The widget is still mounted | A disposed controller is installed into the UI | Check lifecycle and `mounted` |
| Route is popped during initialization | The future can update state safely | `setState()` after dispose | Check `mounted` before publishing |

The failure is easy to miss in a fresh launch. It appears when initialization overlaps a lifecycle transition, which is why a simulator smoke test that only opens the page once is insufficient.

## A controller boundary that survives resume

Keep the controller nullable. `null` is an intentional “camera is being rebuilt” state, not an error. The `_openFuture` guard prevents two resume events from starting two native sessions at once.

```dart
import 'dart:async';

import 'package:camera/camera.dart';
import 'package:flutter/material.dart';

class CameraPage extends StatefulWidget {
  const CameraPage({super.key});

  @override
  State<CameraPage> createState() => _CameraPageState();
}

class _CameraPageState extends State<CameraPage>
    with WidgetsBindingObserver {
  CameraController? _controller;
  CameraDescription? _description;
  Future<void>? _openFuture;
  Object? _error;
  bool _appIsActive = true;

  @override
  void initState() {
    super.initState();
    WidgetsBinding.instance.addObserver(this);
    unawaited(_openCamera());
  }

  @override
  void didChangeAppLifecycleState(AppLifecycleState state) {
    if (state == AppLifecycleState.inactive ||
        state == AppLifecycleState.paused) {
      _appIsActive = false;
      final old = _controller;
      _controller = null;
      if (mounted) setState(() {});
      if (old != null) unawaited(old.dispose());
      return;
    }

    if (state == AppLifecycleState.resumed) {
      _appIsActive = true;
      unawaited(_openCamera());
    }
  }

  Future<void> _openCamera() {
    return _openFuture ??= _openCameraOnce().whenComplete(() {
      _openFuture = null;
    });
  }

  Future<void> _openCameraOnce() async {
    _description ??= (await availableCameras()).first;
    final next = CameraController(
      _description!,
      ResolutionPreset.medium,
      enableAudio: false,
    );

    try {
      await next.initialize();
      if (!mounted || !_appIsActive) {
        await next.dispose();
        return;
      }

      final old = _controller;
      setState(() {
        _error = null;
        _controller = next;
      });
      if (old != null) await old.dispose();
    } catch (error) {
      await next.dispose();
      if (mounted) {
        setState(() => _error = error);
      }
    }
  }

  @override
  void dispose() {
    WidgetsBinding.instance.removeObserver(this);
    final controller = _controller;
    if (controller != null) unawaited(controller.dispose());
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    final controller = _controller;
    if (_error != null) {
      return Center(child: Text('Camera unavailable: $_error'));
    }
    if (controller == null || !controller.value.isInitialized) {
      return const Center(child: CircularProgressIndicator());
    }
    return CameraPreview(controller);
  }
}
```

The important ordering is `setState(() => _controller = next)` only after `initialize()` succeeds. The old controller is disposed after the new one is published, so the preview does not briefly point at an object that has already been released. If your app cannot tolerate two native sessions even briefly, dispose `old` before `setState`, accepting a short blank frame during the swap.

## Traps that remain after adding the observer

Do not call `initialize()` again on a controller that was disposed. A `CameraController` represents one native session; resume means constructing a replacement. Also avoid putting `availableCameras()` in every resume callback. Cache the selected `CameraDescription`, and show a specific error when the device has no usable camera.

The sample rethrows initialization errors because the example has no error surface. Production code should catch that error at the page boundary and render a retry action. Do not silently keep the old controller: after an inactive transition it is precisely the resource that may no longer be valid.

Use this checklist before release:

- Open the preview, press Home, and return several times.
- Start a phone call or permission flow while the preview is visible.
- Pop the route while `initialize()` is still pending.
- Test a device with only a front-facing camera and a denied camera permission.
- Verify that `CameraPreview` is not built until `value.isInitialized` is true.

The `camera` package documentation and changelog were checked on 2026-09-30. The lifecycle rule is package guidance, while the race guard in this post is application code: it is a defensive boundary for asynchronous initialization, not a claim that every device-specific camera error has one cause.

Sources: [camera package documentation](https://pub.dev/packages/camera) and [camera changelog](https://pub.dev/packages/camera/changelog).
