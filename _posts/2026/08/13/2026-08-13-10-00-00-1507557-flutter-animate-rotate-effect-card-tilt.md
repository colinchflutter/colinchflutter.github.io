---
layout: post
title: "flutter_animate RotateEffect - Add Subtle Tilt Feedback to Flutter Cards"
description: "Learn how to use flutter_animate RotateEffect to add controlled tilt feedback to Flutter cards, buttons, and state changes without a manual AnimationController."
date: 2026-08-13
tags: [flutter_animate, animation, performance, Flutter]
comments: true
share: true
---

![flutter_animate RotateEffect tilting a Flutter card between resting and active states](https://colinchflutter.github.io/assets/images/flutter-animate-rotate-effect-card-tilt.png)

The `RotateEffect` in `flutter_animate` is a good fit for small interaction feedback: a card can lean a few degrees when selected, a retry button can acknowledge a tap, or an empty-state illustration can gently rotate into place. The useful range is surprisingly small. A tilt around `0.04` to `0.10` radians usually feels intentional; a larger value quickly starts to look like a page transition.

## Why a tiny rotation works better

I initially treated rotation like a normal entrance animation and used a large angle. The result was technically correct but visually noisy, especially beside a `ScaleEffect`. For a card selection state, the rotation should communicate “this changed” without competing with the label or tap target.

| Use case | Practical angle | Duration | Motion goal |
| --- | ---: | ---: | --- |
| Selected card | `0.05` rad | 180 ms | Small state acknowledgement |
| Press feedback | `-0.04` → `0` rad | 120 ms | Quick physical response |
| Decorative entrance | `-0.10` → `0` rad | 350 ms | Noticeable but calm arrival |

The package accepts the angle in radians, not degrees. That is the first place I lost time when tuning the effect. `4°` is approximately `0.07` radians.

## Add RotateEffect to a Flutter card

This example keeps the animation declarative. The `Animate` widget owns the effect, while the `ValueKey` makes the state transition predictable when the selected item changes.

```dart
import 'package:flutter/material.dart';
import 'package:flutter_animate/flutter_animate.dart';

class SelectableCard extends StatelessWidget {
  const SelectableCard({
    required this.id,
    required this.selected,
    required this.onTap,
    super.key,
  });

  final String id;
  final bool selected;
  final VoidCallback onTap;

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: onTap,
      child: Animate(
        key: ValueKey('$id-$selected'),
        effects: [
          RotateEffect(
            begin: selected ? -0.05 : 0.02,
            end: selected ? 0.02 : 0.0,
            curve: Curves.easeOutBack,
            duration: 180.ms,
          ),
          ScaleEffect(
            begin: const Offset(0.98, 0.98),
            end: const Offset(1, 1),
            duration: 180.ms,
          ),
        ],
        child: Card(
          color: selected ? Colors.indigo.shade50 : Colors.white,
          child: ListTile(
            title: Text('Option $id'),
            trailing: Icon(
              selected ? Icons.check_circle : Icons.circle_outlined,
            ),
          ),
        ),
      ),
    );
  }
}
```

The rotation is applied around the widget’s center. That matters when the card sits close to a clipped parent: the corners can move outside the original bounds even though the layout size has not changed. Give the card a little horizontal padding, and avoid putting a strongly tilted child directly inside a tight `ClipRRect`.

## Choosing between replay and state-driven animation

For a persistent selected state, rebuilding `Animate` with a key is simple and readable. For a one-off tap acknowledgement, use a stable `Animate` and replay the controller instead. Mixing both approaches can make an effect fire twice.

| Situation | Better approach | Common mistake |
| --- | --- | --- |
| Selection changes | State-dependent `begin`/`end` | Replaying every parent rebuild |
| Button tap | Controller replay | Creating a new controller per tap |
| List item update | Stable item key | Letting list indexes identify cards |

Keep the angle modest on low-end devices as well. Rotation itself is inexpensive, but stacking it with blur, shadows, and several simultaneously rebuilding list rows can make the frame budget harder to read. I would start with `RotateEffect` plus one scale or color effect, then profile before adding more.

## A short checklist

- Pass radians, and convert degrees before tuning.
- Keep interaction tilt near `0.05`–`0.10` radians.
- Use stable keys when list items can reorder.
- Leave room for transformed corners near clipped parents.
- Pair rotation with one supporting effect instead of a full effect stack.

`RotateEffect` is most effective when users barely notice the implementation. A restrained tilt gives Flutter cards a physical response while keeping the content readable and the animation code small.
