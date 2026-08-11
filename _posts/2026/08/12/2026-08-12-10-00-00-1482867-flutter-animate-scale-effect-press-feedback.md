---
layout: post
title: "flutter_animate ScaleEffect - Build Natural Flutter Press Feedback"
description: "Learn how to use flutter_animate ScaleEffect for Flutter button press feedback, with state handling, alignment fixes, and reusable code."
date: 2026-08-12
tags: [flutter_animate, animation, performance, Android, iOS]
comments: true
share: true
---

![flutter_animate ScaleEffect creating press feedback on a Flutter action card](https://colinchflutter.github.io/assets/images/flutter-animate-scale-effect-press-feedback.png)

`flutter_animate`'s `ScaleEffect` is a small effect with a surprisingly large impact on perceived responsiveness. A button that shrinks to 96% while it is pressed feels connected to the user's finger, but the same animation can look broken when its alignment or state trigger is wrong.

## The basic press interaction

The useful mental model is simple: keep the widget at its normal size when `pressed` is false, then animate toward a slightly smaller scale while it is true. `target` is convenient here because the effect follows a state value instead of requiring a separate `AnimationController`.

```dart
class PressableCard extends StatefulWidget {
  const PressableCard({super.key, required this.onTap});

  final VoidCallback onTap;

  @override
  State<PressableCard> createState() => _PressableCardState();
}

class _PressableCardState extends State<PressableCard> {
  bool pressed = false;

  void setPressed(bool value) {
    if (pressed != value) setState(() => pressed = value);
  }

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: widget.onTap,
      onTapDown: (_) => setPressed(true),
      onTapUp: (_) => setPressed(false),
      onTapCancel: () => setPressed(false),
      child: Card(
        child: Padding(
          padding: const EdgeInsets.all(20),
          child: Text('Save changes').animate(
            target: pressed ? 1 : 0,
          ).scale(
            begin: const Offset(1, 1),
            end: const Offset(.96, .96),
            duration: 90.ms,
            curve: Curves.easeOut,
          ),
        ),
      ),
    );
  }
}
```

The scale is attached to the text in this short example, but a real card usually animates a wrapper around the entire visual surface. Otherwise the label shrinks while the card edge stays still, which reads as a rendering glitch rather than touch feedback.

## Alignment decides where the card moves

Scaling happens around the widget's alignment. The default center is usually right for a button. A card inside a row or a tile that appears anchored to one edge may need an explicit alignment:

```dart
Card(
  child: content,
).animate(target: pressed ? 1 : 0).scale(
  begin: const Offset(1, 1),
  end: const Offset(.97, .97),
  alignment: Alignment.center,
  duration: 110.ms,
)
```

| Use case | Scale range | Practical result |
|---|---:|---|
| Primary button | 1.00 → 0.96 | Clear, immediate press |
| Dense list tile | 1.00 → 0.98 | Less visual movement |
| Large feature card | 1.00 → 0.97 | Noticeable without feeling bouncy |

The key is not the exact percentage. The motion should confirm the input without changing the layout footprint or making adjacent content jump.

## Traps that show up in real apps

`onTapUp` is not enough. If the pointer leaves the hit area, Flutter can call `onTapCancel`, so always reset the state there. Without it, the card can remain permanently compressed.

Also avoid putting the animated widget behind a changing key. A new key recreates the animation and can replay the entrance from `begin`, even when the user only changed unrelated screen state. Keep the key stable unless replacing the content is intentional.

For accessibility, keep the feedback short and subtle. A 90–120 ms scale transition works well for touch; a longer 300 ms transition makes every tap feel delayed. The effect is visual feedback, not a replacement for a larger semantic hit target.

## Short checklist

- Animate the complete visual surface, not only its label.
- Reset on both `onTapUp` and `onTapCancel`.
- Start with 96–98% scale for touch feedback.
- Set `alignment` when the visual appears edge-anchored.
- Keep the widget key stable during ordinary rebuilds.

`ScaleEffect` earns its place when it communicates state quickly. The best result is almost invisible: users simply feel that the Flutter interface responds at the exact moment they touch it.
