---
layout: post
title: "flutter_animate ShimmerEffect — skeleton loading screens without the shimmer package"
description: "Build polished skeleton loading UIs using flutter_animate's built-in ShimmerEffect. Covers looping setup, card skeletons, color customization, and a common iOS rendering gotcha."
date: 2026-06-25
tags: [flutter_animate, animation, Flutter, performance]
comments: true
share: true
---

# flutter_animate ShimmerEffect — skeleton loading screens without the shimmer package

![Skeleton loading screen on a mobile app](https://images.unsplash.com/photo-1555774698-0b77e0d5fac6?w=800&q=80)

Short answer: `flutter_animate` ships `ShimmerEffect` out of the box. You don't need the `shimmer` package at all — and the flutter_animate version composes cleanly with the rest of your animation chain.

I had the `shimmer` package in the pubspec for two years before realizing flutter_animate already had this. Deleted the dependency, rebuilt the skeleton screens in about 20 minutes.

## What ShimmerEffect actually does

It sweeps a highlight gradient across the widget — the classic "content is loading" shimmer. What makes the flutter_animate version different is that it's just another `Effect`, so you can:

- Sequence it after other effects (`.fade().shimmer()`)
- Control it with the same adapter system as everything else
- Layer it with `.animate(onPlay: (c) => c.repeat())` for infinite looping

The `shimmer` package does one thing well but you end up with a separate widget tree and no easy way to compose with fade-ins or slides.

## Basic setup — looping skeleton card

The key is `onPlay: (c) => c.repeat()`. Without that the shimmer plays once and stops.

```dart
Container(
  width: 300,
  height: 80,
  decoration: BoxDecoration(
    color: Colors.grey.shade300,
    borderRadius: BorderRadius.circular(8),
  ),
)
  .animate(onPlay: (c) => c.repeat())
  .shimmer(duration: 1200.ms, color: Colors.white54)
```

That's it. The `color` is the highlight sweep color — `Colors.white54` gives a subtle look on light backgrounds.

## Building a realistic skeleton card

Real skeletons have multiple placeholder shapes. The trick is to use a `Column` with placeholder containers and apply `animate().shimmer()` to the whole thing — not each piece individually. One sweep, one `AnimationController`.

```dart
class SkeletonCard extends StatelessWidget {
  const SkeletonCard({super.key});

  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: const EdgeInsets.all(16),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          // Avatar placeholder
          Row(
            children: [
              _box(40, 40, circular: true),
              const SizedBox(width: 12),
              Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  _box(120, 14),
                  const SizedBox(height: 6),
                  _box(80, 12),
                ],
              ),
            ],
          ),
          const SizedBox(height: 16),
          // Content lines
          _box(double.infinity, 14),
          const SizedBox(height: 8),
          _box(double.infinity, 14),
          const SizedBox(height: 8),
          _box(200, 14),
        ],
      ),
    )
        .animate(onPlay: (c) => c.repeat())
        .shimmer(
          duration: 1400.ms,
          color: Colors.white60,
          angle: 15, // tilt the sweep gradient slightly
        );
  }

  Widget _box(double width, double height, {bool circular = false}) {
    return Container(
      width: width,
      height: height,
      decoration: BoxDecoration(
        color: Colors.grey.shade300,
        borderRadius: BorderRadius.circular(circular ? height / 2 : 4),
      ),
    );
  }
}
```

The `angle` parameter rotates the shimmer sweep — 15 degrees looks more natural than perfectly horizontal.

## Swapping skeleton → real content

The pattern I use: show `SkeletonCard` when data is null, `RealCard` when it arrives. Wrap the swap in a `crossFade` so it doesn't just pop in.

```dart
data == null
    ? const SkeletonCard()
    : RealCard(data: data!)
        .animate()
        .fadeIn(duration: 300.ms)
```

Don't bother with `AnimatedSwitcher` here — flutter_animate's `.fadeIn()` is less ceremony.

## Color customization for dark mode

`Colors.white54` looks fine on a light grey card, but on dark theme you want a lighter shimmer relative to a darker base.

```dart
final isDark = Theme.of(context).brightness == Brightness.dark;

_skeletonBase.animate(onPlay: (c) => c.repeat()).shimmer(
  duration: 1200.ms,
  color: isDark ? Colors.white12 : Colors.white60,
)
```

The base card color should also adapt:

```dart
color: isDark ? Colors.grey.shade800 : Colors.grey.shade300,
```

## The iOS rendering gotcha

On iOS, if your skeleton card sits inside a `ListView` and you scroll while the shimmer is playing, you sometimes get a brief flicker. This happens because the gradient repaint doesn't clip correctly when the widget goes partially offscreen.

Fix: wrap the shimmer widget in `RepaintBoundary`.

```dart
RepaintBoundary(child: SkeletonCard())
```

This isolates the shimmer's repaint from the scroll layer. Tested on iOS 17, Flutter 3.22 — the flicker disappears. On Android it doesn't matter either way, but `RepaintBoundary` is cheap so I add it regardless.

## Combining shimmer with a fade-in entrance

If the skeleton itself needs to appear gradually (useful when navigating to a new screen), chain a fade before the shimmer:

```dart
SkeletonCard()
  .animate(onPlay: (c) => c.repeat())
  .fadeIn(duration: 200.ms)
  .shimmer(duration: 1400.ms, color: Colors.white54, delay: 200.ms)
```

The `delay: 200.ms` on shimmer waits for the fade to finish before the sweep starts. `ThenEffect` also works here — [covered in the previous post]({% post_url 2026-06-24-16-00-00-396507-flutter-animate-theneffect-sequencing %}).

---

Next up: `ValueNotifierAdapter` — driving flutter_animate from any `ValueNotifier` or `ChangeNotifier`, which is useful when your animation state lives in a Riverpod provider or a Bloc stream.

**References**
- [flutter_animate on pub.dev](https://pub.dev/packages/flutter_animate)
- [ShimmerEffect API docs](https://pub.dev/documentation/flutter_animate/latest/flutter_animate/ShimmerEffect-class.html)
