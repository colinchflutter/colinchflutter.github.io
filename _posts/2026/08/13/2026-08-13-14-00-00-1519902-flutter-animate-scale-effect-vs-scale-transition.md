---
layout: post
title: "flutter_animate ScaleEffect vs ScaleTransition - Choosing the Right Flutter Press Animation"
description: "Compare flutter_animate ScaleEffect with Flutter's ScaleTransition for press feedback, repeatable motion, and controller-driven animations."
date: 2026-08-13
tags: [flutter_animate, animation, performance, testing]
comments: true
share: true
---

![Flutter animation timeline showing a card scaling through three stages](https://images.unsplash.com/photo-1555066931-4365d14bab8c?w=1200&q=80)

*The useful comparison is not which API is shorter, but whether the animation owns its timeline or reacts to one.*

If a Flutter card only needs a small press or entrance animation, `flutter_animate`'s `ScaleEffect` is usually the better fit. If the scale must stay synchronized with drag progress, a gesture, or several other widgets, Flutter's `ScaleTransition` gives you the control you actually need. I used to reach for a controller too early; the extra state made a simple tap interaction harder to reset and test.

## The same motion with two different owners

`ScaleEffect` owns the animation timeline inside the widget chain. The code stays close to the visual intent:

```dart
class ProductCard extends StatelessWidget {
  const ProductCard({super.key});

  @override
  Widget build(BuildContext context) {
    return Card(
      child: const Padding(
        padding: EdgeInsets.all(20),
        child: Text('Wireless keyboard'),
      ),
    ).animate().scale(
      begin: const Offset(.96, .96),
      end: const Offset(1, 1),
      duration: 280.ms,
      curve: Curves.easeOutBack,
    );
  }
}
```

This is a good default for an entrance animation. There is no `AnimationController`, no `TickerProviderStateMixin`, and no listener to dispose. The effect can also be chained with `.fadeIn()` or `.slideY()` without manually calculating when each phase starts.

The trade-off appears when the animation must follow external progress. A `ScaleTransition` exposes the scale animation directly:

```dart
class PressableCard extends StatefulWidget {
  const PressableCard({super.key});

  @override
  State<PressableCard> createState() => _PressableCardState();
}

class _PressableCardState extends State<PressableCard>
    with SingleTickerProviderStateMixin {
  late final AnimationController _controller = AnimationController(
    vsync: this,
    duration: const Duration(milliseconds: 160),
  );

  late final Animation<double> _scale = Tween<double>(
    begin: 1,
    end: .96,
  ).animate(CurvedAnimation(
    parent: _controller,
    curve: Curves.easeOut,
  ));

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTapDown: (_) => _controller.forward(),
      onTapUp: (_) => _controller.reverse(),
      onTapCancel: _controller.reverse,
      child: ScaleTransition(
        scale: _scale,
        child: const Card(
          child: Padding(
            padding: EdgeInsets.all(20),
            child: Text('Press me'),
          ),
        ),
      ),
    );
  }
}
```

The controller version is longer, but it handles cancellation and reversal explicitly. That matters for a finger that leaves the card before release. A one-shot `ScaleEffect` is not a replacement for gesture progress; it is a concise timeline for a widget that should animate when mounted or replayed.

## Decision table

| Requirement | Better choice | Reason |
| --- | --- | --- |
| Card entrance on page load | `ScaleEffect` | Short chain, no controller state |
| Tap down and tap up feedback | `ScaleTransition` | Forward and reverse are explicit |
| Scale tied to drag percentage | `ScaleTransition` | External progress can drive the controller |
| Scale plus fade and slide sequence | `ScaleEffect` | One readable effect timeline |
| Shared progress across several widgets | `ScaleTransition` | Multiple animations can use one controller |

## Two traps that look harmless

First, do not rebuild an animated widget with a changing key just to replay a press effect. That can restart the entire effect chain and reset child state. For a real replay, keep the animation trigger deliberate.

Second, avoid using `ScaleEffect` for layout changes. Scaling paints a widget at a different visual size but does not reserve a new layout size. If neighboring content must move, use a layout animation such as `AnimatedSize` and add scale only as a visual layer.

My rule is simple: use `ScaleEffect` when the motion is part of the widget's presentation, and use `ScaleTransition` when another event owns the timing. That boundary keeps small interactions declarative without blocking the controller-level behavior that advanced gestures require.

### Quick takeaways

- `ScaleEffect` is the clean choice for one-shot and chained Flutter animations.
- `ScaleTransition` wins when gestures, scrubbing, reversal, or shared progress matter.
- Neither API changes layout size; scaling is a paint-level transformation.
