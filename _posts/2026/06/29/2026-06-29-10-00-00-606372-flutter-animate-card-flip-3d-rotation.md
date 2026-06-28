---
layout: post
title: "flutter_animate Card Flip — 3D Rotation, ScaleX Trick, and State-Driven Toggle"
description: "Build a 3D card flip animation in Flutter using flutter_animate — covering the ScaleX midpoint swap, Matrix4 perspective trick, and state-driven toggle with working code examples."
date: 2026-06-29
tags: [flutter_animate, animation, Flutter, CustomPainter, state_management]
comments: true
share: true
---

# flutter_animate Card Flip — 3D Rotation, ScaleX Trick, and State-Driven Toggle

![Playing cards fanned out on a dark surface](https://images.unsplash.com/photo-1502680390469-be75c86b636f?w=800&q=80)

Card flip is one of those animations that every developer thinks will take 20 minutes and ends up consuming an afternoon. The widget swap has to happen at exactly the right frame, the 3D perspective needs a `Matrix4` trick that isn't in any obvious place in the docs, and if you try to animate front and back independently, they desync in ways that are very visible to users.

Here's the approach that actually works with flutter_animate — two methods, depending on how much visual fidelity you need.

## Why the naive approach breaks

The first instinct is to rotate the whole widget 180° using `RotateEffect`:

```dart
// Looks wrong — you see the front mirrored, not a back face
CardWidget()
  .animate(target: _flipped ? 1.0 : 0.0)
  .rotate(end: 0.5) // 0.5 = 180°
```

This does rotate, but it just shows the front face mirrored. There's no "back side" — you only have one widget. You need two widgets and a synchronized swap at the 90° midpoint. That's the core of every card flip implementation.

## Method 1: ScaleX midpoint swap (simpler, works 90% of the time)

This fakes a 3D flip using a 2D scale. Shrink the front to zero on the X axis (first half), swap widgets, then scale the back from zero to full (second half). No `Matrix4` needed.

```dart
class FlipCard extends StatefulWidget {
  final Widget front;
  final Widget back;

  const FlipCard({required this.front, required this.back, super.key});

  @override
  State<FlipCard> createState() => _FlipCardState();
}

class _FlipCardState extends State<FlipCard> {
  bool _flipped = false;
  bool _showBack = false;

  void _onTap() {
    setState(() => _flipped = !_flipped);
    Future.delayed(300.ms, () {
      if (mounted) setState(() => _showBack = _flipped);
    });
  }

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: _onTap,
      child: _showBack
          ? widget.back
              .animate(key: const ValueKey('back'))
              .scaleX(begin: 0, end: 1, duration: 300.ms, curve: Curves.easeOut)
          : widget.front
              .animate(
                key: const ValueKey('front'),
                target: _flipped ? 1.0 : 0.0,
              )
              .scaleX(end: 0, duration: 300.ms, curve: Curves.easeIn),
    );
  }
}
```

The `Future.delayed(300.ms, ...)` is the swap trigger. It fires at exactly the moment the front card hits scaleX = 0, then the back card takes over and scales back out.

One catch: if the user taps rapidly, the `Future.delayed` fires after state has already changed again. Add a `_isAnimating` flag to block re-taps during the transition:

```dart
bool _isAnimating = false;

void _onTap() {
  if (_isAnimating) return;
  _isAnimating = true;
  setState(() => _flipped = !_flipped);
  Future.delayed(600.ms, () {
    if (mounted) {
      setState(() {
        _showBack = _flipped;
        _isAnimating = false;
      });
    }
  });
}
```

![Two-sided card widget showing front and back faces](https://images.unsplash.com/photo-1516321318423-f06f85e504b3?w=800&q=80)

## Method 2: True 3D flip with Matrix4

For a genuine 3D perspective flip — where the card visually recedes as it rotates — you need `Matrix4.rotationY()` with a perspective entry. flutter_animate's `CustomEffect` is the entry point.

```dart
import 'dart:math';
import 'package:flutter_animate/flutter_animate.dart';

class CardFlip3DEffect extends CustomEffect {
  const CardFlip3DEffect({
    super.delay,
    super.duration,
    super.curve,
  });

  @override
  Widget build(
    BuildContext context,
    Widget child,
    AnimationController controller,
    EffectEntry entry,
  ) {
    final animation = buildAnimation(controller, entry);

    return AnimatedBuilder(
      animation: animation,
      child: child,
      builder: (context, child) {
        final angle = animation.value * pi; // 0 → π
        final transform = Matrix4.identity()
          ..setEntry(3, 2, 0.001) // perspective — without this it looks flat
          ..rotateY(angle);

        return Transform(
          transform: transform,
          alignment: Alignment.center,
          child: child,
        );
      },
    );
  }
}
```

Using it with two overlaid widgets in a `Stack`:

```dart
class FlipCard3D extends StatefulWidget {
  final Widget front;
  final Widget back;
  const FlipCard3D({required this.front, required this.back, super.key});

  @override
  State<FlipCard3D> createState() => _FlipCard3DState();
}

class _FlipCard3DState extends State<FlipCard3D> {
  bool _flipped = false;

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: () => setState(() => _flipped = !_flipped),
      child: SizedBox(
        width: 280,
        height: 180,
        child: Stack(
          children: [
            // Front face: rotates 0° → 90° then disappears
            Visibility(
              visible: !_flipped,
              maintainState: true,
              maintainAnimation: true,
              maintainSize: true,
              child: widget.front
                  .animate(target: _flipped ? 1.0 : 0.0)
                  .custom(
                    duration: 300.ms,
                    curve: Curves.easeIn,
                    builder: (context, value, child) {
                      final angle = value * (pi / 2); // 0 → 90°
                      return Transform(
                        transform: Matrix4.identity()
                          ..setEntry(3, 2, 0.001)
                          ..rotateY(angle),
                        alignment: Alignment.center,
                        child: child,
                      );
                    },
                  ),
            ),
            // Back face: starts at -90° and rotates to 0°, delayed by 300ms
            Visibility(
              visible: _flipped,
              maintainState: true,
              maintainAnimation: true,
              maintainSize: true,
              child: widget.back
                  .animate(target: _flipped ? 1.0 : 0.0)
                  .custom(
                    delay: 300.ms,
                    duration: 300.ms,
                    curve: Curves.easeOut,
                    builder: (context, value, child) {
                      final angle = (1 - value) * (pi / 2); // 90° → 0°
                      return Transform(
                        transform: Matrix4.identity()
                          ..setEntry(3, 2, 0.001)
                          ..rotateY(-angle), // negative = opposite direction
                        alignment: Alignment.center,
                        child: child,
                      );
                    },
                  ),
            ),
          ],
        ),
      ),
    );
  }
}
```

The `setEntry(3, 2, 0.001)` line is the thing that took me the longest to find. Without it the 3D rotation looks completely flat — the card just squishes like it's being pressed. That single line adds perspective so it actually looks like something rotating in space.

`maintainState: true` + `maintainAnimation: true` on `Visibility` is also critical. Without it, the hidden widget loses its animation state and snaps when it becomes visible again.

## Wiring up the card faces

A concrete example with styled front and back:

```dart
FlipCard3D(
  front: Container(
    decoration: BoxDecoration(
      borderRadius: BorderRadius.circular(16),
      gradient: const LinearGradient(
        colors: [Color(0xFF667eea), Color(0xFF764ba2)],
        begin: Alignment.topLeft,
        end: Alignment.bottomRight,
      ),
    ),
    padding: const EdgeInsets.all(24),
    child: const Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        Text('•••• •••• •••• 4242',
            style: TextStyle(color: Colors.white, fontSize: 18, letterSpacing: 2)),
        Spacer(),
        Row(
          mainAxisAlignment: MainAxisAlignment.spaceBetween,
          children: [
            Text('COLIN DEV', style: TextStyle(color: Colors.white70)),
            Text('12/28', style: TextStyle(color: Colors.white70)),
          ],
        ),
      ],
    ),
  ),
  back: Container(
    decoration: BoxDecoration(
      borderRadius: BorderRadius.circular(16),
      color: const Color(0xFF2d2d2d),
    ),
    child: Column(
      children: [
        const SizedBox(height: 32),
        Container(height: 48, color: Colors.black),
        const SizedBox(height: 16),
        Padding(
          padding: const EdgeInsets.symmetric(horizontal: 24),
          child: Row(
            mainAxisAlignment: MainAxisAlignment.end,
            children: [
              Container(
                width: 80, height: 32,
                color: Colors.white,
                alignment: Alignment.center,
                child: const Text('CVV', style: TextStyle(fontSize: 12)),
              ),
            ],
          ),
        ),
      ],
    ),
  ),
)
```

## Common pitfalls

**Back face shows mirrored text.** If your back widget has text or asymmetric content, it'll appear flipped when the rotation passes 90°. Fix: wrap the back widget in `Transform(transform: Matrix4.rotationY(pi), ...)` to pre-flip it — then the two rotations cancel out during the animation.

```dart
// Pre-flip the back content so it reads correctly
Transform(
  transform: Matrix4.rotationY(pi),
  alignment: Alignment.center,
  child: BackContentWidget(),
)
```

**Animation doesn't reverse cleanly.** When `_flipped` goes back to `false`, the `.animate(target: 0.0)` should reverse the transition. If it snaps instead of reversing, check that both `Visibility` widgets have `maintainAnimation: true`. The animation controller needs to stay alive even when the widget is hidden.

**Depth looks wrong on low-end devices.** The `0.001` perspective value works for typical card sizes (200–300px). For very large or very small cards, you may need to tune it. Smaller cards might need `0.002`, larger ones might look better at `0.0008`.

---

Next up: wrapping flutter_animate with `go_router` page transitions — adding enter/exit animations to navigation without duplicating animation code across routes.

**References:**
- [flutter_animate on pub.dev](https://pub.dev/packages/flutter_animate)
- [Flutter Transform widget docs](https://api.flutter.dev/flutter/widgets/Transform-class.html)
- [Matrix4 perspective explained (StackOverflow)](https://stackoverflow.com/questions/54309434/flutter-3d-flip-animation)
