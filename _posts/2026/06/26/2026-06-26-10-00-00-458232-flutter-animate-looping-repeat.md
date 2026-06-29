---
layout: post
title: "flutter_animate Looping Animations — Pulse, Spinner, and Breathing Effects with repeat"
description: "How to build looping animations in flutter_animate using onPlay and repeat. Covers pulse buttons, spinning loaders, breathing indicators, and cleanly stopping loops with a saved controller reference."
date: 2026-06-26
tags: [flutter_animate, animation, Flutter, performance]
comments: true
share: true
---

![Looping animation loading spinner UI Flutter](https://images.unsplash.com/photo-1518432031352-d6fc5c10da5a?w=800&q=80)

Looping animations are everywhere — loading spinners, pulsing icons, breathing status dots. flutter_animate makes them trivial to set up, but the "stop it cleanly" part trips people up every time. Here's the pattern.

## The Core: `onPlay` with `controller.repeat()`

flutter_animate exposes the underlying `AnimationController` via callbacks. For a loop, use `onPlay`:

```dart
Icon(Icons.refresh)
  .animate(onPlay: (controller) => controller.repeat())
  .rotate(duration: 1.seconds, curve: Curves.linear)
```

`onPlay` fires once when the animation starts. Calling `controller.repeat()` inside it keeps it going forever. The `curve: Curves.linear` on the rotate matters — any easing curve on a loop looks jerky at the wrap point.

For a ping-pong loop (plays forward, then backward, repeat):

```dart
.animate(onPlay: (controller) => controller.repeat(reverse: true))
```

## Pulse Button

A pulsing icon for "pay attention to this":

```dart
class PulseButton extends StatelessWidget {
  const PulseButton({super.key});

  @override
  Widget build(BuildContext context) {
    return Icon(Icons.notifications, size: 32, color: Colors.red)
      .animate(onPlay: (controller) => controller.repeat(reverse: true))
      .scale(
        begin: const Offset(1.0, 1.0),
        end: const Offset(1.15, 1.15),
        duration: 700.ms,
        curve: Curves.easeInOut,
      )
      .fade(begin: 0.7, end: 1.0, duration: 700.ms, curve: Curves.easeInOut);
  }
}
```

The `reverse: true` means it scales up to 1.15, then back to 1.0, endlessly. Without it the scale jumps from 1.15 back to 1.0 instantly at each loop restart — looks like a glitch.

## Spinning Loader

```dart
SizedBox(
  width: 24,
  height: 24,
  child: CircularProgressIndicator(strokeWidth: 2),
)
.animate(onPlay: (controller) => controller.repeat())
.rotate(duration: 1.2.seconds, curve: Curves.linear)
```

Turns out `CircularProgressIndicator` already has its own internal animation, so this is overkill. Use this pattern when you want a *custom* spinner:

![Custom Flutter spinner with flutter_animate rotate loop](https://images.unsplash.com/photo-1558618666-fcd25c85cd64?w=800&q=80)

```dart
Icon(Icons.autorenew, size: 28, color: Colors.blue)
  .animate(onPlay: (controller) => controller.repeat())
  .rotate(duration: 900.ms, curve: Curves.linear)
```

## Breathing Indicator

Common for "connected" or "live" status dots:

```dart
Container(
  width: 12,
  height: 12,
  decoration: const BoxDecoration(
    color: Colors.green,
    shape: BoxShape.circle,
  ),
)
.animate(onPlay: (controller) => controller.repeat(reverse: true))
.scaleXY(
  begin: 0.75,
  end: 1.25,
  duration: 1.4.seconds,
  curve: Curves.easeInOut,
)
.fade(begin: 0.5, end: 1.0, duration: 1.4.seconds)
```

The `scaleXY` shorthand scales both axes equally. It's cleaner than `scale(begin: Offset(0.75, 0.75), end: Offset(1.25, 1.25))` — same result.

## Stopping the Loop Cleanly

Here's where people get burned: if the widget disposes while a loop is running, Flutter will throw a `setState` called after dispose error — or worse, it'll just be silent and leak the animation.

The fix: save a reference to the controller via `onInit`, then stop it in `dispose`.

```dart
class _LiveStatusState extends State<LiveStatus> {
  AnimationController? _controller;

  @override
  void dispose() {
    _controller?.stop();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Container(
      width: 12,
      height: 12,
      decoration: const BoxDecoration(
        color: Colors.green,
        shape: BoxShape.circle,
      ),
    )
    .animate(
      onInit: (controller) => _controller = controller,
      onPlay: (controller) => controller.repeat(reverse: true),
    )
    .scaleXY(begin: 0.75, end: 1.25, duration: 1.4.seconds, curve: Curves.easeInOut)
    .fade(begin: 0.5, end: 1.0, duration: 1.4.seconds);
  }
}
```

`onInit` fires before `onPlay` and gives you the controller before any animation starts. Storing it and calling `.stop()` in dispose is all you need. Don't bother with `.dispose()` on the controller itself — flutter_animate owns that lifecycle.

## Stopping a Loop Based on State

Sometimes you want the loop to stop when something changes — e.g., upload finishes, connection drops:

```dart
class _UploadIndicatorState extends State<UploadIndicator> {
  AnimationController? _controller;

  @override
  void didUpdateWidget(UploadIndicator oldWidget) {
    super.didUpdateWidget(oldWidget);
    if (widget.isComplete && !oldWidget.isComplete) {
      _controller?.stop();
    }
  }

  @override
  Widget build(BuildContext context) {
    return Icon(Icons.cloud_upload, size: 28)
      .animate(
        onInit: (controller) => _controller = controller,
        onPlay: (controller) {
          if (!widget.isComplete) controller.repeat();
        },
      )
      .rotate(duration: 1.seconds, curve: Curves.linear);
  }
}
```

The check `if (!widget.isComplete)` in `onPlay` guards against starting the loop when the widget rebuilds in an already-complete state. `didUpdateWidget` catches the transition mid-life.

## The `count` Parameter

If you want exactly N loops, not infinite:

```dart
// flutter_animate adds a .loop() extension on AnimationController
.animate(onPlay: (controller) => controller.loop(count: 3))
.shake(duration: 400.ms)
```

`controller.loop(count: 3)` is flutter_animate's extension — plays 3 full cycles then stops. Useful for "attention shake" on a form error.

## Performance Note

Infinite loops always have a cost. A single looping animation is fine. A `ListView` with 50 items each doing a loop is going to hurt. In list contexts, only animate items that are actually visible — use `AutomaticKeepAliveClientMixin` or stop animations when the item scrolls offscreen.

A debug check: in development, set `Animate.restartOnHotReload = true` in your `main()`. Loops restart on every hot reload, so you can iterate on timing without restarting the app.

---

Next up: combining `flutter_animate` with hero transitions — entrance animations that hand off to Flutter's built-in Hero widget.

**References**
- [flutter_animate pub.dev](https://pub.dev/packages/flutter_animate)
- [flutter_animate GitHub](https://github.com/gskinner/flutter_animate)
- [AnimationController.repeat docs](https://api.flutter.dev/flutter/animation/AnimationController/repeat.html)
