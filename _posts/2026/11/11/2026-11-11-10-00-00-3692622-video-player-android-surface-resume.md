---
layout: post
title: "video_player Android Surface - Fix Flutter Video Frames Disappearing After Resume"
description: "Recover Flutter video_player frames that turn white or black after Android background, ads, or navigation by choosing the right view type and rebuilding safely."
date: 2026-11-11
tags: [video_player, Android, performance, debugging, Flutter]
comments: true
share: true
---

![Flutter video_player Android surface lifecycle](https://pub.dev/static/img/pub-dev-icon.svg)

If a Flutter `video_player` keeps playing audio but loses its picture after Android resumes, treat it as a rendering-surface problem first. This fix is most useful for Android 11+ apps that show interstitial ads, open another Activity, or use Impeller/platform views. It does not apply to a normal network failure, and switching every player to `platformView` can introduce navigation lag or composition bugs.

## The failure pattern

The controller can remain initialized while the native surface behind `VideoPlayer` has been reset. That produces a particularly confusing symptom: `isPlaying` is still true, the progress position advances, but the widget is white, black, or frozen on its last frame.

Flutter's Android surface guidance says a surface may be destroyed while an app is suspended and must be initialized again when it becomes available. A matching `video_player` issue reports the same sequence on Android 14 and 15, including the case where an interstitial ad leaves audio running while the video frame never returns.

| Symptom | Likely boundary | First check |
| --- | --- | --- |
| Audio continues, frame is black | Surface was recreated | `video_player` and Flutter versions |
| Frame returns after a short delay | Surface recreation race | Reproduce after home/app switch |
| Black frame only after an ad | Activity focus + surface reset | Test with the ad SDK disabled |
| Navigation becomes slow after workaround | Platform-view composition cost | Compare texture and platform view |

The official `video_player` API exposes `viewType`. The default remains `VideoViewType.textureView`; `VideoViewType.platformView` is an optional alternative on supported platforms. It is a diagnostic and compatibility choice, not a universal performance upgrade.

## A reproducible controller factory

Keep the view-type decision in one place so a release experiment does not require editing every screen. The following example uses the platform view only for an Android build where the texture path is known to lose frames.

```dart
import 'package:flutter/foundation.dart';
import 'package:video_player/video_player.dart';
import 'package:video_player_platform_interface/video_player_platform_interface.dart';

VideoPlayerController createController(Uri uri) {
  final usePlatformView =
      !kIsWeb && defaultTargetPlatform == TargetPlatform.android;

  return VideoPlayerController.networkUrl(
    uri,
    viewType: usePlatformView
        ? VideoViewType.platformView
        : VideoViewType.textureView,
  );
}
```

Do not blindly ship that condition. Platform views can cost more during transitions and scrolling, and the package documentation warns that correctness varies by platform. Start with a feature flag or a device-model allowlist if the failure is limited to a renderer, OS release, or ad flow.

## Rebuild after a real lifecycle boundary

Changing `viewType` affects creation, so changing it on an existing controller does nothing. When the player is known to be unusable after resume, dispose the old controller and create a new one. Preserve the URL and desired playback state outside the controller.

```dart
class PlayerState extends State<Player> with WidgetsBindingObserver {
  VideoPlayerController? _controller;
  Duration _resumeAt = Duration.zero;
  bool _wasPlaying = false;

  @override
  void initState() {
    super.initState();
    WidgetsBinding.instance.addObserver(this);
    _open();
  }

  Future<void> _open() async {
    final controller = createController(widget.uri);
    _controller = controller;
    await controller.initialize();
    if (!mounted || controller != _controller) return;
    await controller.seekTo(_resumeAt);
    if (_wasPlaying) await controller.play();
    setState(() {});
  }

  @override
  void didChangeAppLifecycleState(AppLifecycleState state) {
    if (state == AppLifecycleState.inactive ||
        state == AppLifecycleState.paused) {
      final controller = _controller;
      if (controller == null || !controller.value.isInitialized) return;
      _resumeAt = controller.value.position;
      _wasPlaying = controller.value.isPlaying;
    }
    if (state == AppLifecycleState.resumed &&
        _controller?.value.hasError == true) {
      final old = _controller;
      _controller = null;
      old?.dispose();
      _open();
    }
  }

  @override
  void dispose() {
    WidgetsBinding.instance.removeObserver(this);
    _controller?.dispose();
    super.dispose();
  }
}
```

This sample only rebuilds when the controller reports an error. If the device shows a silent black surface with no error, add a product-level “reinitialize player” trigger after the ad callback or route return. Avoid rebuilding on every `resumed` event: Android can emit focus transitions without destroying the surface, and unnecessary initialization causes buffering and duplicate network work.

## Verification checklist

- Test Android 11, 14, and 15 with the same video URL.
- Background the app while playing, return, and repeat after an interstitial ad.
- Record `value.position`, `isPlaying`, `isInitialized`, and `hasError` before and after resume.
- Compare `textureView` and `platformView` on the affected device.
- Test route pop, fast scrolling, subtitles, fullscreen, and disposal.
- Keep a fallback when the selected view type is unsupported or introduces visible composition defects.

The Flutter surface migration guidance documents `SurfaceProducer` callbacks for plugin authors, while application developers should normally consume the fixed package rather than patch `video_player_android` locally. For `video_player` 2.10 and newer, the public `viewType` option makes a controlled comparison possible. A source fork may hide the symptom today but turns the next plugin upgrade into a second migration.

The practical rule is simple: separate playback state from rendering state. When audio and position continue but pixels disappear, preserve the former, recreate the latter, and choose the view type with measured evidence from the affected Android path.

Sources: [video_player package](https://pub.dev/packages/video_player), [video_player controller API](https://github.com/flutter/packages/blob/main/packages/video_player/video_player/lib/video_player.dart), [Flutter Android surface migration](https://docs.flutter.dev/release/breaking-changes/android-surface-plugins), and [video_player resume issue #161030](https://github.com/flutter/flutter/issues/161030) (checked 2026-11-11).
