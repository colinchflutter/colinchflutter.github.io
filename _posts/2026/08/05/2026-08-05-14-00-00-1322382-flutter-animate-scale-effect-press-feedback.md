---
layout: post
title: "flutter_animate ScaleEffect - Add Flutter Press Feedback Without Layout Shifts"
description: "Learn how to use flutter_animate ScaleEffect and target state to build responsive Flutter press feedback without changing layout dimensions."
date: 2026-08-05
tags: [flutter_animate, animation, performance, Flutter]
comments: true
share: true
---

![Flutter press feedback with flutter_animate ScaleEffect](https://images.unsplash.com/photo-1551650975-87deedd944c3?w=800&q=80)

*The useful detail is that the button gets smaller visually while its layout slot stays exactly the same.*

`flutter_animate`'s `ScaleEffect` is a good fit for press feedback when the interaction should feel immediate but must not push neighboring widgets around. The effect uses a transform, so a card or button can shrink to `0.96` during a press without changing the constraints used by its parent. The `target` value turns a boolean interaction state into a reversible animation.

## A pressable card with `ScaleEffect`

Keep the pointer state outside the animation. The widget owns whether the pointer is down; `flutter_animate` only renders the transition between the two states.

```dart
class PressableCard extends StatefulWidget {
  const PressableCard({super.key});

  @override
  State<PressableCard> createState() => _PressableCardState();
}

class _PressableCardState extends State<PressableCard> {
  bool _pressed = false;

  void _setPressed(bool value) {
    if (_pressed != value) setState(() => _pressed = value);
  }

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTapDown: (_) => _setPressed(true),
      onTapUp: (_) => _setPressed(false),
      onTapCancel: () => _setPressed(false),
      onTap: () => debugPrint('Card tapped'),
      child: Card(
        child: ListTile(
          leading: const Icon(Icons.bolt),
          title: const Text('Run analysis'),
          subtitle: const Text('Tap for instant visual feedback'),
        ),
      ).animate(target: _pressed ? 1 : 0).scale(
        begin: const Offset(1, 1),
        end: const Offset(.96, .96),
        duration: 90.ms,
        curve: Curves.easeOut,
      ),
    );
  }
}
```

The animation moves toward `end` while `_pressed` is `1`, then returns to `begin` when the finger or mouse is released. `onTapCancel` matters on mobile: a drag, interrupted gesture, or pointer leaving the hit area must restore the normal scale too.

## Why the layout does not jump

`ScaleEffect` applies a visual transform around the widget's center. It does not animate `width`, `height`, `padding`, or the `Card`'s constraints. That makes it safer for rows, grids, and compact toolbars than changing the size of the child itself.

| Approach | Layout space | Press release | Common problem |
|---|---|---|---|
| `ScaleEffect` | Stable | Smooth and reversible | Can clip if the parent is tight |
| `AnimatedContainer` size | Changes | Smooth | Nearby children move |
| `Transform.scale` manually | Stable | You manage it | More controller/state code |

If a scaled card looks clipped, check an ancestor using `ClipRect`, a tight `SizedBox`, or a custom sliver. The scale itself is not a layout change, but the transformed pixels still need room to paint. A scale below `1` usually avoids overflow; a pressed value such as `1.04` can expose the clipping immediately.

## Avoid replaying the entrance animation

Do not combine this interaction with an unconditional `.animate().fadeIn()` on a widget that rebuilds frequently. A rebuild caused by unrelated state can make the card look as if it is entering again. Use a stable key when the parent replaces list data, and keep the press animation's `target` tied only to the pointer state.

For touch feedback, a short duration around 80–120 milliseconds is enough. Longer values make the press feel delayed, while a very small scale difference such as `0.99` may be invisible. The practical starting point is `0.96` for a card and `0.94` for a prominent button, then adjust after testing on a real device.

The pattern is small but dependable: state describes the gesture, `ScaleEffect` handles the interpolation, and the transform preserves the layout contract.

- [flutter_animate on pub.dev](https://pub.dev/packages/flutter_animate)
- [ScaleEffect API documentation](https://pub.dev/documentation/flutter_animate/latest/flutter_animate/ScaleEffect-class.html)
