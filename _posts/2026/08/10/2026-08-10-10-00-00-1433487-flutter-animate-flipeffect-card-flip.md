---
layout: post
title: "flutter_animate FlipEffect - Build a Flutter Card Flip Without a Custom Controller"
description: "Learn how to use flutter_animate FlipEffect for Flutter card flips, including perspective, state keys, tap handling, and reverse animation traps."
date: 2026-08-10
tags: [flutter_animate, animation, performance, Android, iOS]
comments: true
share: true
---

![Flutter card flip animation using flutter_animate](/assets/images/flutter-animate-curves-motion.png)

`flutter_animate`'s `FlipEffect` is enough for a practical Flutter card flip. It adds the 2.5D rotation and timeline management, so the widget does not need a hand-written `AnimationController`. The part that still needs care is not the rotation itself. It is choosing the rotation axis, keeping the card readable at 90 degrees, and forcing a fresh animation when the card's state changes.

## The smallest useful FlipEffect

The effect accepts `begin`, `end`, `direction`, and `perspective`. Values are measured against a 180-degree flip: `0.5` is 90 degrees and `1.0` is 180 degrees. A horizontal flip is a good default for a playing card because the card rotates around its vertical axis.

```dart
import 'package:flutter/material.dart';
import 'package:flutter_animate/flutter_animate.dart';

class FlipCard extends StatefulWidget {
  const FlipCard({super.key});

  @override
  State<FlipCard> createState() => _FlipCardState();
}

class _FlipCardState extends State<FlipCard> {
  bool _showBack = false;

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: () => setState(() => _showBack = !_showBack),
      child: (_showBack ? _back() : _front())
          .animate(key: ValueKey(_showBack))
          .flipH(
            begin: 0,
            end: 1,
            duration: 600.ms,
            curve: Curves.easeInOut,
            perspective: 1.2,
          ),
    );
  }

  Widget _front() => _face(
        color: Colors.indigo,
        icon: Icons.lock_outline,
        label: 'Tap to reveal',
      );

  Widget _back() => _face(
        color: Colors.teal,
        icon: Icons.check_circle_outline,
        label: 'Revealed',
      );

  Widget _face({
    required Color color,
    required IconData icon,
    required String label,
  }) {
    return SizedBox(
      width: 220,
      height: 140,
      child: Card(
        color: color,
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Icon(icon, color: Colors.white, size: 36),
            const SizedBox(height: 12),
            Text(label, style: const TextStyle(color: Colors.white)),
          ],
        ),
      ),
    );
  }
}
```

The `ValueKey` is deliberate. Flutter otherwise sees the same animated subtree after `setState`. The boolean changes the child, but the existing `Animate` state can already be at the end of its controller. Giving each face a different key makes the flip start from zero on every tap.

## What the direction actually changes

`flipH()` rotates around the vertical axis, while `flipV()` rotates around the horizontal axis. The choice should follow the physical metaphor rather than the screen orientation.

| Effect | Axis | Good fit | Common visual mistake |
| --- | --- | --- | --- |
| `flipH()` | vertical | card reveal, account details | using a very wide card, which makes the edge feel slow |
| `flipV()` | horizontal | ticket, expandable panel | making the panel too tall, which exaggerates the hinge |

For a vertical flip, the same state pattern becomes:

```dart
final face = _showBack ? _back() : _front();

return face
    .animate(key: ValueKey(_showBack))
    .flipV(begin: 0, end: 1, duration: 550.ms);
```

Do not pass degrees such as `180` to `end`. The effect's scale is based on a half-turn, so `180` would request 180 half-turns. It is one of those values that looks plausible in code but produces a completely broken animation.

## Preventing the upside-down second face

A full `0 → 1` turn shows the replacement face upside down for half of the timeline. For a real front/back card, that is usually not what the user expects. A simple solution is to animate only a half turn and switch the face at the midpoint. `SwapEffect` is useful when the replacement must happen at a precise timeline point, but the two widgets need separate ownership of their animation.

For many settings panels, the simpler `0 → 1` pattern is still fine because both states use the same orientation and the flip reads as a playful reveal. I use it for small status cards, not for text-heavy content.

## Tap handling and layout stability

Keep the `GestureDetector` outside the animated widget. If the detector is inside a transformed subtree, its hit region can become surprising during the rotation, especially when the card uses a perspective transform or extends beyond its original bounds.

The card also needs a fixed size. A front face containing one line of text and a back face containing several lines can cause the parent layout to resize while the transform is running. Wrap both faces in the same `SizedBox`, or give the parent a constrained size. The flip should change pixels, not the surrounding layout.

If the card is inside a list, use a key based on the item ID, not the list index. Otherwise a reorder can preserve the completed animation state and make a different item appear already flipped.

## When FlipEffect is the wrong tool

`FlipEffect` is a visual transform. It does not automatically manage semantic state, network loading, or a continuously changing child. Choose a different pattern when:

- the content must crossfade rather than rotate;
- the two faces have different heights and the layout must interpolate;
- the animation is controlled by drag progress instead of time;
- the same card needs a custom 3D camera and shadow model.

For drag-controlled motion, an adapter or a regular `AnimationController` gives the gesture a direct position. For a simple one-shot reveal, `FlipEffect` keeps the code much smaller.

## Practical checklist

1. Use the flip scale (`0.5` means 90 degrees and `1.0` means 180 degrees), not degrees.
2. Pick `flipH` for a vertical hinge and `flipV` for a horizontal hinge.
3. Add a `ValueKey` when the front and back are different widget trees.
4. Keep both faces constrained to the same size.
5. Put hit testing around the animation and use stable item IDs in lists.

The useful boundary is clear: `FlipEffect` handles the motion, while the widget state decides which face exists. Keeping those responsibilities separate gives a card flip that replays reliably without a custom controller.

- [flutter_animate FlipEffect API documentation](https://pub.dev/documentation/flutter_animate/latest/flutter_animate/FlipEffect-class.html)
