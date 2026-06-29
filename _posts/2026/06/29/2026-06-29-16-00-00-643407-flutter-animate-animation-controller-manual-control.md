---
layout: post
title: "flutter_animate with AnimationController — Manual Playback, Reverse, and Multi-Widget Sync"
description: "Pass your own AnimationController into flutter_animate to manually trigger playback, reverse mid-animation, and synchronize multiple widgets to a single shared controller."
date: 2026-06-29
tags: [flutter_animate, animation, AnimationController, Flutter, state_management]
comments: true
share: true
---

![Developer coding on multiple monitors with dark IDE](https://images.unsplash.com/photo-1461749280684-dccba630e2f6?w=800&q=80)

By default, flutter_animate plays itself the moment a widget builds. That covers 80% of use cases. The rest — animate on button press, reverse halfway through a gesture, sync three widgets to a single trigger — require bringing your own `AnimationController`.

Here's exactly how to do it without the usual boilerplate headaches.

## autoPlay: false is only half the answer

The first instinct is to just set `autoPlay: false`. That stops the auto-play, but you still can't control anything without also passing a `controller`:

```dart
late AnimationController _controller;

@override
void initState() {
  super.initState();
  _controller = AnimationController(vsync: this, duration: 600.ms);
}

@override
void dispose() {
  _controller.dispose();
  super.dispose();
}

@override
Widget build(BuildContext context) {
  return Column(
    children: [
      MyWidget()
          .animate(autoPlay: false, controller: _controller)
          .slideY(begin: 0.3, end: 0)
          .fade(),
      ElevatedButton(
        onPressed: () => _controller.forward(),
        child: const Text('Animate'),
      ),
    ],
  );
}
```

Two things matter here. First: pass **both** `autoPlay: false` and `controller: _controller`. `autoPlay: false` alone just freezes the animation — you have no way to start it without the controller reference. Second: the controller's `duration` takes priority over flutter_animate's per-effect timing. Set it on the controller.

## Reversing mid-animation

This is where external controllers genuinely shine. A button that scales down when pressed and snaps back on release:

```dart
GestureDetector(
  onTapDown: (_) => _controller.forward(),
  onTapUp: (_) => _controller.reverse(),
  onTapCancel: () => _controller.reverse(),
  child: Container(width: 200, height: 60, color: Colors.blue)
      .animate(autoPlay: false, controller: _controller)
      .scaleXY(begin: 1.0, end: 0.9)
      .fade(begin: 1.0, end: 0.6),
)
```

`reverse()` doesn't reset to the beginning — it plays backward from wherever the controller currently is. A half-pressed tap release snaps back proportionally. That's the feel you want for interactive feedback.

One catch: if `_controller.value` is already at 1.0, calling `forward()` does nothing. Guard against it:

```dart
onTapDown: (_) {
  if (_controller.value < 1.0) _controller.forward();
},
```

I spent a while debugging a button that worked on first press but ignored subsequent presses. The controller had finished and was sitting at 1.0. No error, no warning — just silence.

## Synchronizing multiple widgets to one controller

One `AnimationController` can drive any number of widgets. All of them react to the same timeline:

```dart
@override
Widget build(BuildContext context) {
  return Stack(
    children: [
      // Background overlay dims
      Container(color: Colors.black)
          .animate(autoPlay: false, controller: _controller)
          .fade(begin: 0.0, end: 0.6),

      // Card slides up from bottom
      _buildCard()
          .animate(autoPlay: false, controller: _controller)
          .slideY(begin: 1.0, end: 0, curve: Curves.easeOutCubic),

      // Action buttons fade in slightly delayed
      _buildActions()
          .animate(autoPlay: false, controller: _controller)
          .fade(delay: 200.ms),
    ],
  );
}
```

One `_controller.forward()` call dims the background, slides the card, and fades in the buttons — synchronized, no coordination code.

The `delay: 200.ms` on the third widget still works, but it's relative to the controller's start. If the controller's `duration` is shorter than the effect's delay plus its own duration, the animation gets clipped. Bump the controller duration to cover everything:

```dart
// Make sure this is longer than max(delay + effect duration) across all widgets
_controller = AnimationController(vsync: this, duration: 800.ms);
```

![Data dashboard showing synchronized timelines](https://images.unsplash.com/photo-1551288049-bebda4e38f71?w=800&q=80)

## reset() vs reverse() — not the same thing

```dart
// Plays backward from current position (smooth)
_controller.reverse();

// Snaps to 0.0 instantly (no animation)
_controller.reset();

// Replay from scratch
_controller.reset();
_controller.forward();
```

`reverse()` is for interactive scenarios. `reset()` is for when a widget is dismissed and you want it ready for next time — silently, without animating.

I got caught calling `_controller.reset()` inside `didUpdateWidget` without checking which prop changed. Parent rebuilt for an unrelated reason, controller reset mid-animation, jarring snap. The fix: scope the reset to the specific state transition you care about:

```dart
@override
void didUpdateWidget(MyWidget oldWidget) {
  super.didUpdateWidget(oldWidget);
  if (widget.isVisible == false && oldWidget.isVisible == true) {
    _controller.reverse();
  } else if (widget.isVisible == true && oldWidget.isVisible == false) {
    _controller.forward();
  }
}
```

## When not to bother

Most animations don't need external `AnimationController`. If it plays once on build, drop both `autoPlay` and `controller`. Looping? Use `.animate(onPlay: (c) => c.repeat())`. External controller adds boilerplate — reach for it only when you actually need:

- Gesture-driven playback (hold/release, swipe-driven progress)
- Multiple widgets synced to one trigger
- Programmatic trigger after async work (play after API call returns)
- Reverse or reset from outside the widget

---

Next: flutter_animate in bottom sheets and dialogs — triggering entry and exit animations on overlays that Flutter's routing system doesn't fully own.

**References:**
- [flutter_animate on pub.dev](https://pub.dev/packages/flutter_animate)
- [Flutter AnimationController API](https://api.flutter.dev/flutter/animation/AnimationController-class.html)
- [Flutter animations tutorial](https://docs.flutter.dev/ui/animations/tutorial)
- [flutter_animate page transitions with go_router]({% post_url 2026-06-29-14-00-00-631062-flutter-animate-go-router-page-transitions %})
