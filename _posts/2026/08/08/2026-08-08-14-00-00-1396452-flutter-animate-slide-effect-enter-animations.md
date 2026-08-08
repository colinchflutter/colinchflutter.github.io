---
layout: post
title: "flutter_animate SlideEffect - Build Flutter Enter Animations Without Layout Jumps"
description: "Learn how to use flutter_animate SlideEffect for clean Flutter enter animations, staggered lists, directional transitions, and stable layouts."
date: 2026-08-08
tags: [flutter_animate, animation, performance, ListView]
comments: true
share: true
---

![A Flutter card sliding into a stable interface layout with flutter_animate](/assets/images/flutter-animate-slide-effect-layout.png)

*The moving card is animated into an existing slot while the surrounding layout stays fixed.*

`SlideEffect` is the practical choice when a Flutter widget should appear from an offset position without making the widgets around it move. The important detail is that it paints the child at a translated position; it does not reserve a second layout slot for the animation. That makes it useful for notification cards, list rows, onboarding panels, and small pieces of state feedback.

## The basic SlideEffect

The effect is added to the same `.animate()` chain used by the rest of `flutter_animate`. The offset is expressed as a fraction of the child's own size, so `Offset(-1, 0)` starts one child-width to the left.

```dart
import 'package:flutter/material.dart';
import 'package:flutter_animate/flutter_animate.dart';

class NoticeCard extends StatelessWidget {
  const NoticeCard({super.key});

  @override
  Widget build(BuildContext context) {
    return Card(
      child: const ListTile(
        leading: Icon(Icons.check_circle),
        title: Text('Backup completed'),
        subtitle: Text('Your files are safe.'),
      ),
    ).animate().slideX(
      begin: -0.35,
      end: 0,
      duration: 420.ms,
      curve: Curves.easeOutCubic,
    );
  }
}
```

The convenience method `slideX()` creates a `SlideEffect` for a horizontal transition. Use `slideY()` for a vertical entrance, or use `SlideEffect` directly when both axes need separate values.

```dart
Widget panel(int index) {
  return Panel(index: index).animate(
    delay: (index * 55).ms,
  ).slide(
    begin: const Offset(0, 0.18),
    end: Offset.zero,
    duration: 360.ms,
    curve: Curves.easeOut,
  );
}
```

The stagger is deliberately small. A delay of 55 milliseconds is enough to reveal order in a short list, while larger values make a screen feel like it is waiting for each row.

## Slide without moving the layout

There is a common mistake here: conditionally adding a new widget and expecting the slide to animate when the parent is rebuilt. `SlideEffect` animates the child that already exists. If the child is removed immediately, there is nothing left to animate out.

For a persistent slot, keep the widget in the tree and change its content or use an explicit key for state changes:

```dart
AnimatedBuilder(
  animation: controller,
  builder: (context, child) {
    return child!.animate(
      target: controller.value,
    ).slideX(
      begin: -0.2,
      end: 0,
      duration: 260.ms,
    );
  },
  child: const NoticeCard(),
)
```

For a one-time entrance, the simpler `.animate().slideX()` chain is enough. For a repeated state transition, control the target or rebuild a keyed child so the effect is not unexpectedly replayed on every parent update.

## Choosing the offset and curve

| Use case | Begin offset | Curve | Why |
| --- | ---: | --- | --- |
| Toast from the edge | `Offset(1, 0)` | `easeOutCubic` | Quick arrival, soft stop |
| Form section reveal | `Offset(0, 0.12)` | `easeOut` | Keeps movement subtle |
| Staggered list rows | `Offset(0, 0.18)` | `easeOut` | Shows order without drama |
| Error replacement | `Offset(-0.08, 0)` | `easeInOut` | Small directional cue |

Large offsets are tempting, but they often expose the widget outside its intended clip area. Wrap the animation in `ClipRect` when the entering content must stay inside a card or panel. Also avoid combining a large `SlideEffect` with `MoveEffect` unless the two motions have clearly different jobs; the resulting distance is easy to misread and harder to test.

## A few checks that prevent rough motion

- Keep the animated widget's final size stable. If its height changes during the same frame, the animation can look like a layout jump even though the slide itself is correct.
- Use stable keys in animated lists. Without them, Flutter may reuse a row's state and make the wrong item appear to slide.
- Respect reduced-motion preferences for screens where the transition is decorative. A short fade or no entrance motion is usually better than forcing a full-distance slide.
- Test the first frame and the settled frame. The widget should be fully visible at `end: Offset.zero`, with no residual transform or unexpected overflow.

`SlideEffect` works best as a small spatial cue, not as a substitute for layout animation. Reserve the slot, keep the offset modest, and let the surrounding UI remain still. That combination gives Flutter enter animations a clear direction without making the interface feel unstable.
