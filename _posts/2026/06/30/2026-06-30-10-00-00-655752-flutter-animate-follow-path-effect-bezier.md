---
layout: post
title: "flutter_animate FollowPathEffect — Animating Widgets Along Custom Bezier Paths"
description: "flutter_animate has no built-in FollowPathEffect, but CustomEffect paired with Flutter's PathMetric lets you move any widget along a smooth bezier curve in under 30 lines."
date: 2026-06-30
tags: [flutter_animate, animation, CustomPainter, performance]
comments: true
share: true
---

![Flutter animation light trails following a curved path](https://images.unsplash.com/photo-1558618666-fcd25c85cd64?w=800&q=80)

flutter_animate has no built-in `FollowPathEffect`. I searched the docs, scanned the source — it's not there. What it does have is `CustomEffect`, and that's all you need to make any widget follow a bezier curve you draw yourself.

The use cases are more common than you'd think: map pin drops, onboarding spotlight tours that arc between UI elements, progress indicators that chase a curved track, game characters following a patrol route. `MoveEffect` only goes in a straight line, so as soon as you need a curve, you're writing a custom effect.

## Why MoveEffect Falls Short

`MoveEffect` translates a widget from one position to another linearly. You can chain them:

```dart
Icon(Icons.circle)
  .animate()
  .moveX(end: 100, duration: 400.ms)
  .then()
  .moveY(end: -80, duration: 400.ms);
```

But chaining creates sharp corners at each segment. There's no way to tell flutter_animate "follow this arc." For a smooth curve you need Flutter's `Path` + `PathMetric`.

## PathMetric: Sampling a Path at Any Progress Value

Flutter's `PathMetric` is the key. Once you compute metrics on a `Path`, you can call `getTangentForOffset(distance)` to get both position and angle at any point along the path:

```dart
final path = Path()
  ..moveTo(0, 0)
  ..cubicTo(60, -80, 140, -80, 200, 0);

final metrics = path.computeMetrics().toList();
final metric = metrics.first;

// At 50% progress:
final tangent = metric.getTangentForOffset(metric.length * 0.5);
print(tangent?.position); // somewhere along the arc
print(tangent?.angle);    // radians, direction of travel
```

`CustomEffect` in flutter_animate hands you a `value` from 0.0 to 1.0 as the animation runs. Multiply that by `metric.length` and you get position along the path. That's the entire trick.

## The Implementation

```dart
Widget buildFollowPath({
  required Widget child,
  required Path path,
  bool rotate = false,
  Duration duration = const Duration(milliseconds: 1000),
  Curve curve = Curves.linear,
}) {
  return child.animate().custom(
    duration: duration,
    curve: curve,
    builder: (context, value, child) {
      final metrics = path.computeMetrics().toList();
      if (metrics.isEmpty) return child;

      final metric = metrics.first;
      final tangent = metric.getTangentForOffset(metric.length * value);
      if (tangent == null) return child;

      Widget result = Transform.translate(
        offset: Offset(tangent.position.dx, tangent.position.dy),
        child: child,
      );

      if (rotate) {
        result = Transform.rotate(angle: tangent.angle, child: result);
      }

      return result;
    },
  );
}
```

The `rotate: true` flag uses `tangent.angle` to tilt the widget so it faces the direction of travel — essential for arrows, cars, or any directional icon.

## Defining the Path

Standard Flutter `Path` API. Cubic bezier segments give the smoothest results:

```dart
// Arc up then back down — good for a ball bounce
final bouncePath = Path()
  ..moveTo(0, 0)
  ..cubicTo(60, -100, 140, -100, 200, 0);

// S-curve — good for ribbon or snake effect
final sCurvePath = Path()
  ..moveTo(0, 0)
  ..cubicTo(80, -80, 120, 80, 200, 0);

// Drop from above with slight overshoot
final dropPath = Path()
  ..moveTo(0, -180)
  ..cubicTo(10, -180, -10, 20, 0, 0);
```

Coordinates are relative to the widget's **current position on screen**. `moveTo(0, 0)` means "start where the widget is right now." If you want it to end 200px to the right, your path should end at `(200, 0)`.

## Real Examples

**Map pin that drops and lands:**

```dart
final pinDrop = Path()
  ..moveTo(0, -160)
  ..cubicTo(5, -160, -5, 10, 0, 0);

Icon(Icons.location_on, color: Colors.red, size: 40)
  .animate()
  .custom(
    duration: 600.ms,
    curve: Curves.easeOut,
    builder: (context, value, child) {
      final metric = pinDrop.computeMetrics().first;
      final t = metric.getTangentForOffset(metric.length * value);
      if (t == null) return child;
      return Transform.translate(offset: t.position, child: child);
    },
  )
  .scaleXY(begin: 0.6, end: 1.0, duration: 300.ms, curve: Curves.bounceOut);
```

**Rocket following an arc trajectory:**

```dart
final flightPath = Path()
  ..moveTo(0, 0)
  ..cubicTo(100, -200, 300, -200, 400, 0);

buildFollowPath(
  child: const Text('🚀', style: TextStyle(fontSize: 28)),
  path: flightPath,
  rotate: true,
  duration: 1800.ms,
  curve: Curves.easeInOut,
);
```

![Widget path animation coordinate system diagram](https://images.unsplash.com/photo-1633356122544-f134324a6cee?w=800&q=80)

## The Coordinate Space Gotcha

The thing that tripped me up first: path coordinates are in local widget space, not screen space. If your widget sits at `(200, 400)` on the screen and your path starts at `moveTo(0, 0)`, the animation starts exactly at the widget's current position — not at screen origin `(0, 0)`.

If you need to move a widget from position A to position B in screen coordinates, you have to compute the delta:

```dart
// Widget A is at screenPos, widget B is at targetPos
// Both obtained via RenderBox.localToGlobal(Offset.zero)
final delta = targetPos - screenPos;

final path = Path()
  ..moveTo(0, 0)
  ..cubicTo(
    delta.dx * 0.3, -80,
    delta.dx * 0.7, -80,
    delta.dx, delta.dy,
  );
```

## Looping

Set `repeat: true` via `AnimateList` or pass the controller directly:

```dart
Icon(Icons.circle, size: 12)
  .animate(onPlay: (controller) => controller.repeat())
  .custom(
    duration: 2.seconds,
    curve: Curves.linear,   // linear is critical for loops — easing looks wrong at the seam
    builder: (context, value, child) {
      final metric = loopPath.computeMetrics().first;
      final t = metric.getTangentForOffset(metric.length * value);
      if (t == null) return child;
      return Transform.translate(offset: t.position, child: child);
    },
  );
```

Use `Curves.linear` for loops. Ease-in/out looks smooth on a single run, but when it restarts it snaps from slow → fast at the seam and looks broken.

## Onboarding Spotlight Tour

The use case I've gotten the most mileage from: a highlight ring that arcs between two buttons in an onboarding flow, making it obvious which control to tap next.

```dart
final tourPath = Path()
  ..moveTo(0, 0)
  ..cubicTo(-50, -70, 150, -70, 120, 0);

Container(
  width: 56,
  height: 56,
  decoration: BoxDecoration(
    shape: BoxShape.circle,
    border: Border.all(color: Colors.amber, width: 3),
  ),
)
  .animate()
  .custom(
    duration: 800.ms,
    curve: Curves.easeInOut,
    builder: (context, value, child) {
      final metric = tourPath.computeMetrics().first;
      final t = metric.getTangentForOffset(metric.length * value);
      if (t == null) return child;
      return Transform.translate(offset: t.position, child: child);
    },
  )
  .fadeIn(duration: 200.ms)
  .then(delay: 400.ms)
  .fadeOut(duration: 200.ms);
```

Chain the `CallbackEffect` (see {% post_url 2026-06-28-12-00-00-581682-flutter-animate-callback-listen-effect-lifecycle %}) at the end to trigger the next step when the animation completes. Combined with the `AnimationController` patterns from {% post_url 2026-06-29-16-00-00-643407-flutter-animate-animation-controller-manual-control %}, you can sequence an entire multi-step tour without a state machine.

---

Next up: `ToggleEffect` in flutter_animate — alternating widget states on each animation cycle without resetting back to the beginning.

**References**
- [flutter_animate on pub.dev](https://pub.dev/packages/flutter_animate)
- [Flutter Path class docs](https://api.flutter.dev/flutter/dart-ui/Path-class.html)
- [PathMetric docs](https://api.flutter.dev/flutter/dart-ui/PathMetric-class.html)
- [Tangent class docs](https://api.flutter.dev/flutter/dart-ui/Tangent-class.html)
