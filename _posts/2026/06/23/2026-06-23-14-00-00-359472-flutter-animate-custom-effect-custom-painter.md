---
layout: post
title: "flutter_animate CustomEffect — animating custom-drawn widgets"
description: "flutter_animate's CustomEffect drives a double from 0 to 1 through any curve, letting you animate CustomPainter paths, progress arcs, and hand-drawn shapes without writing an AnimationController."
date: 2026-06-23
tags: [flutter_animate, CustomPainter, animation, Flutter, Dart]
comments: true
share: true
---
![Custom paint animation on canvas](https://images.unsplash.com/photo-1635070041078-e363dbe005cb?w=800&q=80)

`flutter_animate` handles `fadeIn`, `slideX`, `scale` cleanly. But as soon as you have a `CustomPainter` — a progress arc, a hand-drawn path, a signature stroke — you're on your own. The built-in effects don't reach inside a painter. `CustomEffect` is how you fix that.

Short answer: `CustomEffect` gives your painter a `double` that goes from `0.0` to `1.0` through whatever curve and duration you set. Your painter draws based on that value. That's the whole mechanism.

## A minimal CustomPainter to drive

Here's a painter that draws a circular arc from 0 to `progress * 2π`:

```dart
class ArcPainter extends CustomPainter {
  ArcPainter({required this.progress});
  final double progress;

  @override
  void paint(Canvas canvas, Size size) {
    final paint = Paint()
      ..color = Colors.deepPurple
      ..strokeWidth = 6
      ..style = PaintingStyle.stroke
      ..strokeCap = StrokeCap.round;

    final rect = Rect.fromCircle(
      center: size.center(Offset.zero),
      radius: size.shortestSide / 2 - 6,
    );

    canvas.drawArc(rect, -1.5708, 2 * 3.14159 * progress, false, paint);
  }

  @override
  bool shouldRepaint(ArcPainter old) => old.progress != progress;
}
```

The key line: `shouldRepaint` returns `true` only when `progress` actually changed. Skipping this makes the animation render once and stop. Spent 20 minutes on that.

## Wiring CustomEffect

`CustomEffect` takes a builder callback that receives the animation value and returns a widget:

```dart
CustomPaint(
  size: const Size(120, 120),
  painter: ArcPainter(progress: 0),
)
  .animate()
  .custom(
    duration: 1200.ms,
    curve: Curves.easeOut,
    builder: (context, value, child) {
      return CustomPaint(
        size: const Size(120, 120),
        painter: ArcPainter(progress: value),
      );
    },
  )
```

The `value` parameter is the `double` that sweeps 0→1. `child` is the original widget passed in — you can ignore it if you're rebuilding the painter from scratch, or reuse it as an overlay.

## Chaining with other effects

`CustomEffect` plays nicely with the chain. This fades in the container while the arc draws:

```dart
Container(
  width: 140,
  height: 140,
  alignment: Alignment.center,
  child: CustomPaint(
    size: const Size(120, 120),
    painter: ArcPainter(progress: 0),
  ),
)
  .animate()
  .fadeIn(duration: 400.ms)
  .custom(
    delay: 100.ms,
    duration: 1000.ms,
    curve: Curves.easeInOut,
    builder: (context, value, child) => CustomPaint(
      size: const Size(120, 120),
      painter: ArcPainter(progress: value),
    ),
  )
```

The `delay: 100.ms` starts the arc draw slightly after the fade begins.

## The part that doesn't work like you'd expect

`CustomEffect` replaces the widget it's called on. The `child` in the builder is whatever widget was before `.animate()`. If you chain another effect after `.custom(...)`, it wraps the widget returned by your builder, not the original.

This matters when you have a `Stack` or complex widget tree inside the painter host. I had a badge overlaid on the arc — I put it outside the custom effect and it worked fine:

```dart
Stack(
  alignment: Alignment.center,
  children: [
    CustomPaint(
      size: const Size(120, 120),
      painter: ArcPainter(progress: 0),
    )
      .animate()
      .custom(
        duration: 1200.ms,
        curve: Curves.easeOut,
        builder: (context, value, child) => CustomPaint(
          size: const Size(120, 120),
          painter: ArcPainter(progress: value),
        ),
      ),
    Text('${(value * 100).toInt()}%'), // drives separately
  ],
)
```

Except `value` isn't in scope there. To show a percentage counter that matches the arc, you need a `StatefulWidget` that listens to the same controller — or pull out a shared `AnimationController` manually. At that point you're back to the old way for the text portion. Not ideal, but that's just how the package is scoped.

## When to use this over AnimationController

- Quick entry animation on a painter with no other listeners: `CustomEffect` wins
- Multiple widgets sharing the same animation value, or you need fine-grained control: just use `AnimationController`

`flutter_animate` doesn't replace the animation system — it removes boilerplate for the common cases. `CustomEffect` extends that to painters, which covers most dashboard widgets, loaders, and decorative entry sequences.

---

Next: controlling `flutter_animate` playback with `AnimateController` — how to pause, reverse, and replay animations from outside the widget.
