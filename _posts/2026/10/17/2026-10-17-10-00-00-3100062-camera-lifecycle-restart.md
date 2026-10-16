---
layout: post
title: "camera Flutter Lifecycle - Rebuild the Controller Without Resume Races"
description: "Fix Flutter camera preview crashes after backgrounding by owning AppLifecycleState, serializing controller disposal and reinitialization, and testing the resume race explicitly."
date: 2026-10-17
tags: [camera, lifecycle, Android, iOS, performance]
comments: true
share: true
---

<svg xmlns="http://www.w3.org/2000/svg" width="1000" height="260" viewBox="0 0 1000 260" role="img" aria-labelledby="camera-flow-title camera-flow-desc">
  <title id="camera-flow-title">Flutter camera lifecycle restart flow</title>
  <desc id="camera-flow-desc">Inactive disposes the camera controller, resumed creates and initializes a fresh controller.</desc>
  <rect width="1000" height="260" rx="24" fill="#172033"/>
  <text x="42" y="48" fill="#f8fafc" font-family="Arial,sans-serif" font-size="27" font-weight="700">camera controller ownership</text>
  <rect x="42" y="92" width="245" height="112" rx="16" fill="#26364f"/><text x="75" y="130" fill="#bfdbfe" font-family="Arial,sans-serif" font-size="24" font-weight="700">active</text><text x="75" y="165" fill="#e2e8f0" font-family="Arial,sans-serif" font-size="19">preview uses controller</text>
  <path d="M307 148h102m-12-12 16 12-16 12" stroke="#60a5fa" stroke-width="7" fill="none"/>
  <rect x="426" y="92" width="220" height="112" rx="16" fill="#7c2d3d"/><text x="462" y="130" fill="#fecdd3" font-family="Arial,sans-serif" font-size="24" font-weight="700">inactive</text><text x="462" y="165" fill="#ffe4e6" font-family="Arial,sans-serif" font-size="19">dispose + clear ref</text>
  <path d="M666 148h102m-12-12 16 12-16 12" stroke="#5eead4" stroke-width="7" fill="none"/>
  <rect x="795" y="92" width="163" height="112" rx="16" fill="#0f766e"/><text x="828" y="130" fill="#ccfbf1" font-family="Arial,sans-serif" font-size="24" font-weight="700">resumed</text><text x="828" y="165" fill="#ecfeff" font-family="Arial,sans-serif" font-size="18">new controller</text>
</svg>

*Read the flow as ownership: the inactive boundary releases the native resource, and resumed receives a fresh controller.*

The `camera` package is safest when the screen treats every resume as a resource recovery, not as a request to reuse the old `CameraController`. This applies to camera previews that can be covered by a permission sheet, phone call, or another app. It is not needed for a screen that only receives a file from `image_picker`, and it should not keep the camera open while the app is inactive.

## The trap introduced by lifecycle ownership

Since `camera` 0.5.0, the plugin no longer closes and reopens the camera for Android lifecycle changes. The package README puts that responsibility on the app. A controller that worked before the app was backgrounded may therefore be disposed, disconnected, or associated with a stale native camera ID when the user returns.

| Situation | Wrong assumption | Safer decision |
| --- | --- | --- |
| `inactive` | The preview will pause itself | Dispose the controller and clear the reference |
| `resumed` | The old controller can be initialized again | Create a new controller from the saved description |
| Resume during an unfinished initialize | Every async callback still belongs to the screen | Serialize lifecycle work and check `mounted` |
| Route is popped | `dispose()` is only cleanup after the last frame | Unregister the observer and await controller disposal |

The difficult bug is not just a black preview. A fast background/resume sequence can overlap `dispose()` and `initialize()`. The old initialization then completes after the new state has already been installed, producing a platform exception or a `setState()` call for a screen that no longer exists.

## Keep one lifecycle owner

The following state object deliberately uses a small restart queue. The queue is more useful than a boolean such as `_isResuming`, because lifecycle callbacks can arrive while an earlier disposal or initialization is still awaiting the platform.

```dart
class CameraScreenState extends State<CameraScreen>
    with WidgetsBindingObserver {
  CameraController? _controller;
  CameraDescription? _description;
  Future<void> _lifecycleWork = Future<void>.value();

  @override
  void initState() {
    super.initState();
    WidgetsBinding.instance.addObserver(this);
    _description = widget.camera;
    _lifecycleWork = _createController(widget.camera);
  }

  @override
  void didChangeAppLifecycleState(AppLifecycleState state) {
    if (state == AppLifecycleState.inactive) {
      _lifecycleWork = _lifecycleWork.then((_) => _closeController());
    } else if (state == AppLifecycleState.resumed) {
      final description = _description;
      if (description != null) {
        _lifecycleWork = _lifecycleWork.then(
          (_) => _createController(description),
        );
      }
    }
  }

  Future<void> _closeController() async {
    final controller = _controller;
    _controller = null;
    if (mounted) setState(() {});
    await controller?.dispose();
  }

  Future<void> _createController(CameraDescription description) async {
    await _closeController();
    final next = CameraController(description, ResolutionPreset.medium);
    try {
      await next.initialize();
    } catch (_) {
      await next.dispose();
      rethrow;
    }
    if (!mounted) {
      await next.dispose();
      return;
    }
    _controller = next;
    setState(() {});
  }

  @override
  void dispose() {
    WidgetsBinding.instance.removeObserver(this);
    _controller?.dispose();
    super.dispose();
  }
}
```

`_createController()` closes any previous instance before constructing the next one. That makes camera switching and lifecycle recovery share one ownership rule. The `mounted` check prevents a late initialization from publishing a controller after navigation has removed the screen. The `catch` path also disposes a controller whose native initialization failed; otherwise a failed permission or hardware start can leave resources behind.

## Choose the right policy for each state

`inactive` is the conservative boundary recommended by the package example. On some products, a brief transition to `inactive` occurs while a permission dialog is visible. Disposing there means the resume path must recreate the preview, but it avoids holding the camera while the app is not active. If that restart is visually expensive, measure it on the target devices before changing the policy to `paused`; do not assume the Android and iOS callback sequence is identical.

The `camera` package currently defaults Android to the CameraX implementation. If a device-specific regression is tied to CameraX, switching implementations is a compatibility decision with feature and device trade-offs, not a substitute for lifecycle handling. Keep the package version and selected implementation pinned while diagnosing the issue so a dependency update does not change two variables at once.

## A reproducible resume checklist

- Open the preview and wait until `isInitialized` is true.
- Trigger a permission sheet or lock/unlock the device while the preview is visible.
- Send the app to the background, return immediately, and repeat five times.
- Pop the route during initialization and confirm there is no `setState() called after dispose`.
- Deny camera permission, resume, grant permission in Settings, and retry from a fresh controller.
- Switch front/rear cameras during a resume cycle and verify that the description used for recreation is still the intended one.

The package’s official README documents the lifecycle handoff, and its changelog records the breaking behavior from 0.5.0. The [Flutter camera cookbook](https://docs.flutter.dev/cookbook/plugins/picture-using-camera) also requires initialization before preview use and disposal when the screen is removed. Those are separate guarantees: initialization protects the first frame, while lifecycle ownership protects later native resource transitions.

The short rule is: save the `CameraDescription`, dispose on the inactive boundary, serialize recovery, and only publish a newly initialized controller while the screen is mounted. A camera preview is a native resource with a Dart UI attached; treating it like an ordinary immutable widget is what makes the resume race difficult to see.
