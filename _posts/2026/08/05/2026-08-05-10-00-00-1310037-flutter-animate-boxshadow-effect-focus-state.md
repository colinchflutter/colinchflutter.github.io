---
layout: post
title: "flutter_animate BoxShadowEffect - Animate Flutter Focus States Without Layout Shifts"
description: "Use flutter_animate BoxShadowEffect to animate focus and selected states in Flutter without changing layout, hit testing, or card size."
date: 2026-08-05
tags: [flutter_animate, animation, Flutter, accessibility]
comments: true
share: true
---

![Flutter card focus state with an animated shadow using flutter_animate](/assets/images/flutter-animate-curves-motion.png)

*The useful part of a focus animation is the changing boundary around the control, not a card that moves when its shadow grows.*

`flutter_animate`'s `BoxShadowEffect` is a good fit for selected cards, keyboard focus, and drag targets because the shadow is painted outside the content layout. I initially used a larger border and a changing padding value for the same effect. That made a grid jump by a few pixels whenever selection changed. Animating the shadow keeps the card's constraints stable while still giving the state a visible change.

## A focusable card with a stable size

Keep the state in the parent and use `target` to move the effect between its inactive and active values. The child keeps the same padding and dimensions in both states.

```dart
class ChoiceCard extends StatelessWidget {
  const ChoiceCard({
    required this.label,
    required this.selected,
    required this.onTap,
    super.key,
  });

  final String label;
  final bool selected;
  final VoidCallback onTap;

  @override
  Widget build(BuildContext context) {
    final color = Theme.of(context).colorScheme.primary;

    return Semantics(
      button: true,
      selected: selected,
      label: '$label${selected ? ', selected' : ''}',
      child: InkWell(
        onTap: onTap,
        borderRadius: BorderRadius.circular(16),
        child: Card(
          clipBehavior: Clip.antiAlias,
          child: Padding(
            padding: const EdgeInsets.all(20),
            child: Text(label),
          ),
        )
            .animate(target: selected ? 1 : 0)
            .boxShadow(
              begin: const BoxShadow(
                color: Colors.transparent,
                blurRadius: 0,
                spreadRadius: 0,
              ),
              end: BoxShadow(
                color: color.withValues(alpha: .28),
                blurRadius: 14,
                spreadRadius: 2,
              ),
              duration: const Duration(milliseconds: 220),
            ),
      ),
    );
  }
}
```

`target: 0` shows the `begin` shadow and `target: 1` shows the `end` shadow. The important detail is that the `Card` itself does not receive a new border or padding when `selected` changes. That prevents neighboring grid cells from being laid out again just because the visual emphasis changed.

## Focus and selection are different states

A selected card represents application state. Keyboard focus represents where input will go next. They can share a subtle shadow, but they should not be treated as the same boolean.

| State | Visual treatment | Semantic source |
|---|---|---|
| Not selected | Transparent shadow | `selected: false` |
| Selected | Colored, slightly spread shadow | `selected: true` |
| Keyboard focused | Stronger contrast or outline | `FocusNode.hasFocus` |
| Disabled | No animated emphasis | `onTap: null` and disabled semantics |

For a real keyboard focus ring, add an `AnimatedBuilder` around `FocusNode` and change the shadow target from `selected ? 1 : 0` to `focused || selected ? 1 : 0`. Do not remove the focus indication just because the mouse version looks polished; keyboard users need a persistent location cue.

## Traps with animated shadows

- A large `blurRadius` is not free. Dozens of cards with animated shadows can cause extra paint work, especially in a scrolling grid. Keep the blur around 8–16 pixels and profile the actual screen.
- `BoxShadowEffect` does not change hit testing. A shadow outside the card is decoration, so keep the `InkWell` and its semantic label on the actual control.
- Avoid `repeat()` for focus feedback. Focus is a state, not a loading indicator; a looping glow becomes distracting when the user tabs through a form.
- If the shadow is the only selection cue, add a text, icon, color, or semantic change as well. Animation can be skipped or reduced by platform settings.

The practical rule is to animate paint properties when the state change is visual. `BoxShadowEffect` gives a selected or focused Flutter control a clear boundary while its layout, hit area, and surrounding widgets stay put.

- [flutter_animate on pub.dev](https://pub.dev/packages/flutter_animate)
- [Flutter BoxShadow API](https://api.flutter.dev/flutter/painting/BoxShadow-class.html)
