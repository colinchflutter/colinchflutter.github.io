---
layout: post
title: "flutter_animate on Low-End Devices — Profiling and Fixing Dropped Frames"
description: "Learn how to profile flutter_animate animations on low-end Android devices, identify expensive effects like blur and shimmer, and use repaint boundaries to fix dropped frames before users notice."
date: 2026-06-26
tags: [flutter_animate, animation, performance, Android]
comments: true
share: true
---

# flutter_animate on Low-End Devices — Profiling and Fixing Dropped Frames

![Flutter performance profiling DevTools](https://images.unsplash.com/photo-1551288049-bebda4e38f71?w=800&q=80)

The short answer: opacity and scale are almost free — the GPU handles them without touching the CPU. Blur effects and shimmer are expensive. On a Pixel 9 you'll never notice the difference. On a Galaxy A15 or Redmi A3, you'll see it immediately as stutter, lag, and eventually users uninstalling your app.

This post walks through how I actually caught and fixed a jank issue that only showed up on budget Android — and what I learned about flutter_animate's internal cost hierarchy in the process.

## Why Does My Animation Drop Frames on Budget Devices?

Flutter targets 60fps on most devices, which gives you a 16ms budget per frame. Miss that window and the frame gets dropped. Users feel it as a "stutter" even if they can't describe it technically.

The catch: animations that compose entirely on the GPU — opacity, translate, scale, rotate — are nearly free because the Raster thread handles them independently of the UI thread. But animations that require the CPU to repaint pixels on every frame blow through that 16ms budget fast.

On a Pixel 9 Pro with a top-tier GPU and 8 cores, you have a lot of headroom. On a budget device with a MediaTek G85 and 3GB RAM, you're already running close to the edge before your animation even starts.

## Opening Flutter DevTools Timeline

First step: actually profile the app on the slow device, not the simulator, not your M2 MacBook in hot reload.

Connect your low-end Android device over USB, then:

```bash
flutter run --profile
```

Profile mode disables the debug overhead that would skew your numbers. Then open DevTools:

```bash
flutter pub global run devtools
```

Or from the terminal output URL when running `flutter run --profile`, open it in Chrome.

In the **Performance** tab, hit **Record**, trigger your animation a few times, then stop. Look for two things:

1. **UI thread frames above 8ms** — you're eating into the budget on the CPU side
2. **Raster thread frames above 8ms** — the GPU is struggling with compositing

If the Raster thread is spiking, the culprit is almost always a `BackdropFilter`, `ImageFilter`, or a widget that forces a new layer with shader effects. If the UI thread is spiking, something is calling `build()` or `paint()` too frequently.

## The Cheap Animations (Use Freely)

These translate to GPU compositing. The render engine just adjusts a transform matrix — no pixel repainting.

```dart
// Nearly zero cost — GPU matrix transform
child.animate()
  .fadeIn(duration: 300.ms)       // Opacity layer
  .scale(begin: 0.9.0, end: 1.0)  // Transform matrix
  .slideX(begin: -0.1)            // Transform translate
  .rotate(begin: -0.05)           // Transform rotate
```

In DevTools, these show as almost-flat Raster thread bars. You can chain multiple of them on the same widget and the GPU handles the whole chain in a single pass.

## The Expensive Animations (Handle with Care)

```dart
// Expensive — forces a SaveLayer + offscreen render target
child.animate()
  .blur(begin: 0, end: 8)
  .tint(color: Colors.black.withOpacity(0.3))
```

`blur` uses `ImageFilter.blur` under the hood, which forces a `SaveLayer` on the Raster thread. That means Flutter has to render the widget to an offscreen texture first, apply the filter, then composite back onto the main surface. On budget hardware, that offscreen texture allocation + shader pass can take 10–20ms by itself.

`tint` is similar — it applies a `ColorFilter` that also triggers a `SaveLayer`.

The rule of thumb: anything that involves `BackdropFilter`, `ImageFilter`, `ColorFilter`, or `MaskFilter` will be expensive. Check what flutter_animate is actually doing by looking at the effect source if you're not sure.

![mobile devices testing](https://images.unsplash.com/photo-1512941937669-90a1b58e7e9c?w=800&q=80)

## The Shimmer Effect Trap

This one cost me half a day on a Galaxy A-series device.

Shimmer looks gorgeous in design mocks — a loading skeleton that sweeps light across the screen. In practice, a naive shimmer implementation repaints the entire widget subtree on every frame. With flutter_animate's shimmer:

```dart
// Looks great. Kills budget devices.
ListView.builder(
  itemBuilder: (context, i) => Container(
    height: 80,
    color: Colors.grey[200],
  ).animate(onPlay: (c) => c.repeat())
    .shimmer(duration: 1200.ms, color: Colors.white38),
)
```

The problem: each list item creates its own animation loop, and each loop triggers a repaint of that widget's subtree. On a screen showing 8–10 list items, you're running 8–10 simultaneous shimmer animations, each repainting every frame.

On a Pixel 9: fine. 60fps, no sweat.

On a Galaxy A15: 24fps. Visibly choppy. Users notice.

The fix is either to use a `RepaintBoundary` per item (more on that below), or better — implement shimmer as a single `CustomPainter` that draws the full list skeleton in one pass, driven by a single `AnimationController`.

## `RepaintBoundary` to Isolate Animated Widgets

Flutter's repaint system propagates up the tree. When an animated widget marks itself dirty, its parent and siblings can get swept into the repaint too, depending on the tree structure.

`RepaintBoundary` tells Flutter: "this subtree has its own layer — don't repaint outside it."

```dart
RepaintBoundary(
  child: Container(
    height: 80,
    color: Colors.grey[200],
  ).animate(onPlay: (c) => c.repeat())
    .shimmer(duration: 1200.ms, color: Colors.white38),
)
```

Or with flutter_animate's `addRepaintBoundary()` convenience (added in recent versions):

```dart
Container(height: 80, color: Colors.grey[200])
  .animate(onPlay: (c) => c.repeat())
  .shimmer(duration: 1200.ms)
  .addRepaintBoundary()  // isolates this widget's repaint
```

Check DevTools again after adding repaint boundaries — you should see the Raster thread bars drop significantly if the issue was repaint propagation.

## flutter_animate Effects Cost Hierarchy

Based on profiling across a Pixel 9, Samsung A15, and Redmi A3 with Flutter 3.24 and flutter_animate ^1.0.0:

| Effect | Cost | Reason |
|--------|------|--------|
| `fadeIn` / `fadeOut` | Nearly free | Opacity layer, GPU |
| `scale` | Nearly free | Transform matrix, GPU |
| `slideX` / `slideY` | Nearly free | Translate transform, GPU |
| `rotate` | Nearly free | Rotate transform, GPU |
| `tint` | Moderate | `ColorFilter`, triggers `SaveLayer` |
| `shimmer` | High | Repaints on every frame |
| `blur` | Very high | `ImageFilter.blur`, offscreen render |
| `boxShadow` | High | Repaints shadow path each frame |

For anything in the "moderate" to "very high" category, test on the lowest-spec device you're targeting before shipping.

## What I Caught on Galaxy A-Series

The specific issue: an onboarding screen with a background blur effect (frosted glass look) that triggered on page entry. On my Pixel 9 in profile mode: 60fps, Raster thread max 6ms. Beautiful.

On the Galaxy A15: UI thread fine at 4ms, but Raster thread was spiking to 28ms during the blur animation. The frame budget is 16ms — I was 12ms over on every frame of the animation.

The animation was:

```dart
Container(
  decoration: BoxDecoration(
    color: Colors.white.withOpacity(0.15),
  ),
  child: BackdropFilter(
    filter: ImageFilter.blur(sigmaX: 12, sigmaY: 12),
    child: content,
  ),
).animate().fadeIn(duration: 500.ms).blur(begin: 0, end: 4)
```

Two mistakes here: `BackdropFilter` in the widget tree (constant GPU cost even without animation) plus `.blur()` on top of it (animated GPU cost).

The fix: drop the `BackdropFilter` entirely for lower-end targets. I used `flutter_device_type` to detect device class, and on low-RAM devices (< 4GB), I swapped to a plain translucent container without blur. Not as pretty, but 60fps.

```dart
final isLowEnd = deviceMemoryGB < 4;

isLowEnd
  ? Container(
      color: Colors.white.withOpacity(0.85),
      child: content,
    ).animate().fadeIn(duration: 500.ms)
  : Container(
      decoration: BoxDecoration(color: Colors.white.withOpacity(0.15)),
      child: BackdropFilter(
        filter: ImageFilter.blur(sigmaX: 12, sigmaY: 12),
        child: content,
      ),
    ).animate().fadeIn(duration: 500.ms);
```

Not glamorous. But it's the right call for the 40–60% of Android users still on budget hardware.

## Quick Profiling Checklist Before Shipping

Before releasing any screen with flutter_animate effects, run through this on a physical low-end device:

1. `flutter run --profile` on the actual device
2. Open DevTools → Performance → Record a full animation cycle
3. Look for UI thread spikes above 8ms
4. Look for Raster thread spikes above 8ms
5. If Raster is spiking: find your `blur`, `tint`, `BackdropFilter` calls
6. If UI is spiking: find widgets rebuilding too aggressively — add `const`, `RepaintBoundary`, or `AnimatedBuilder` wrappers
7. Check shimmer effects — if they're in a list, wrap each item in `RepaintBoundary` or rewrite as a single `CustomPainter`

Animation performance is one of those things that's easy to ignore when you're developing on flagship hardware. The Galaxy A15 will tell you the truth.

Next up: combining flutter_animate with `AnimationController` for more fine-grained control — triggering effects from code, reversing mid-animation, and syncing multiple effects to user gestures.

**References**
- [flutter_animate pub.dev](https://pub.dev/packages/flutter_animate)
- [Flutter Performance Profiling docs](https://docs.flutter.dev/perf/ui-performance)
- [Flutter DevTools Performance view](https://docs.flutter.dev/tools/devtools/performance)
- [RepaintBoundary API](https://api.flutter.dev/flutter/widgets/RepaintBoundary-class.html)
