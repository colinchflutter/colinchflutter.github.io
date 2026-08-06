---
layout: post
title: "flutter_animate MoveEffect - Animate Flutter Widgets on Two Axes Without Layout Jumps"
description: "Learn how to use flutter_animate MoveEffect to animate Flutter widgets from any X/Y offset, sequence motion, and avoid layout jumps."
date: 2026-08-06
tags: [flutter_animate, animation, performance, Flutter]
comments: true
share: true
---

![Flutter MoveEffect animation moving a status card from an offset](/assets/images/flutter-animate-curves-motion.png)

`MoveEffect` is the right `flutter_animate` choice when a widget should enter from a two-dimensional offset, such as a card sliding in diagonally or a toast arriving from the corner. The important detail is that the offset is visual motion: it does not reserve a second copy of the widget or change the surrounding layout.

## The problem with hand-written slide animations

I first reached for `AnimatedPositioned` when building a floating status card. It worked inside a `Stack`, but the animation became awkward as soon as the same card had to appear in a regular `Column`. The animation needed a parent with position values, and changing those values could also make neighboring content re-layout.

`MoveEffect` keeps the animation attached to the widget itself. The widget is laid out at its final size, then painted at an interpolated offset during the animation.

| Goal | Useful `begin` offset | Visual result |
| --- | --- | --- |
| Enter from the left | `Offset(-40, 0)` | Horizontal slide |
| Enter from below | `Offset(0, 24)` | Upward rise |
| Enter from a corner | `Offset(28, 18)` | Diagonal slide |
| Exit toward the right | `Offset.zero` to `Offset(40, 0)` | Horizontal leave |

The offset uses logical pixels. A value of `Offset(40, 0)` means the widget starts 40 pixels to the right of its final position, not 40 percent of the screen width.

## A reusable two-axis entrance

This example shows a message that arrives from the bottom-right and then fades out after its visible state changes.

```dart
import 'package:flutter/material.dart';
import 'package:flutter_animate/flutter_animate.dart';

class StatusMessage extends StatelessWidget {
  const StatusMessage({super.key, required this.message});

  final String message;

  @override
  Widget build(BuildContext context) {
    return Card(
      margin: const EdgeInsets.symmetric(horizontal: 16, vertical: 8),
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Row(
          children: [
            const Icon(Icons.check_circle, color: Colors.green),
            const SizedBox(width: 12),
            Expanded(child: Text(message)),
          ],
        ),
      ),
    )
        .animate(key: ValueKey(message))
        .move(
          begin: const Offset(28, 18),
          end: Offset.zero,
          duration: 360.ms,
          curve: Curves.easeOutCubic,
        )
        .fadeIn(duration: 220.ms);
  }
}
```

The `ValueKey` matters when this widget is replaced with a new message. Without a changing key, Flutter can preserve the existing animated element and the entrance may not replay when the text changes. The key should represent the event, not a random value generated on every build.

## Chaining movement with `ThenEffect`

For a notification that settles with a small nudge, split the motion into two readable phases instead of using a single complicated curve.

```dart
final animation = notification.animate()
  .move(
    begin: const Offset(0, 30),
    duration: 300.ms,
    curve: Curves.easeOut,
  )
  .then()
  .move(
    begin: const Offset(0, -4),
    end: Offset.zero,
    duration: 90.ms,
    curve: Curves.easeInOut,
  );
```

Here, the second effect starts from the first effect's result. A common mistake is to set the second `begin` to the original 30-pixel offset again, which makes the widget jump backward at the phase boundary.

## What to check before shipping

- Use `MoveEffect` for paint-time motion. Use `AnimatedPositioned` when the layout itself must respond to the position.
- Keep entrance offsets modest. On a compact phone, 16–40 logical pixels usually communicates direction without making the message feel detached.
- Test `MediaQuery.disableAnimations` or the app's reduced-motion policy. A decorative slide should be removable without hiding a required state change.
- If the widget is in a scrolling list, provide stable keys. Otherwise Flutter may reuse an element and animate the wrong row.

The practical rule is simple: let layout determine where the widget belongs, and let `MoveEffect` determine how it reaches that position. That separation makes two-axis motion easy to reuse in banners, validation messages, list rows, and route-level overlays.
