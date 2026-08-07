---
layout: post
title: "flutter_animate AlignEffect - Animate Flutter Layout Alignment Without Jumps"
description: "Learn how to use flutter_animate AlignEffect to animate Flutter widgets between alignments, with target-driven state changes and layout constraints explained."
date: 2026-08-08
tags: [flutter_animate, animation, Flutter, performance]
comments: true
share: true
---

![Flutter widget moving smoothly between top-center and center alignment with flutter_animate](/assets/images/flutter-animate-curves-motion.png)

*The important detail is the bounded canvas: AlignEffect changes where the child sits inside available space, not the child's own size.*

`flutter_animate`'s `AlignEffect` is useful when a widget should move between meaningful layout positions: a login card settling into the center, an empty-state message moving above a button, or a notification pinning itself to the top of a panel. It animates the alignment itself, so the child stays in the layout instead of being pushed around with arbitrary offsets.

## The basic AlignEffect

The effect needs room to show the alignment change. A `Center` or a tightly sized parent hides most of the animation, so give the child a bounded area first.

```dart
SizedBox(
  height: 280,
  width: double.infinity,
  child: ColoredBox(
    color: const Color(0xFF111827),
    child: const _StatusCard()
        .animate()
        .align(
          begin: Alignment.topCenter,
          end: Alignment.center,
          duration: 450.ms,
          curve: Curves.easeOutCubic,
        ),
  ),
)
```

`AlignEffect` wraps the target with `Align` and interpolates from `begin` to `end`. The card moves inside the `SizedBox`; it does not change the height of the surrounding layout. That distinction matters in a `Column`, where a size animation can make every sibling move while an alignment animation only changes the card's position.

## Connecting it to state

For a real UI, set `target` to `0` or `1`. When the value changes, `flutter_animate` reverses the same effect chain without a separate `AnimationController`.

```dart
class StatusPanel extends StatefulWidget {
  const StatusPanel({super.key});

  @override
  State<StatusPanel> createState() => _StatusPanelState();
}

class _StatusPanelState extends State<StatusPanel> {
  bool _expanded = false;

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        SizedBox(
          height: 220,
          width: double.infinity,
          child: _StatusCard().animate(
            target: _expanded ? 1 : 0,
          ).align(
            begin: Alignment.topCenter,
            end: Alignment.center,
            duration: 400.ms,
          ),
        ),
        FilledButton(
          onPressed: () => setState(() => _expanded = !_expanded),
          child: Text(_expanded ? 'Collapse' : 'Center card'),
        ),
      ],
    );
  }
}
```

The first render starts at `begin`, because `_expanded` is false. Tapping the button changes the target to `1`, and the card moves to `end`. A common mistake is to remove the `SizedBox` while testing this. Without extra height, the parent may give the card only its intrinsic size, leaving no visible space for `Alignment.topCenter` and `Alignment.center` to differ.

## Alignment versus MoveEffect

| Choose | Best for | Typical failure |
| --- | --- | --- |
| `AlignEffect` | Moving within a known layout region | Parent has no extra space |
| `MoveEffect` | Pixel-based motion or decorative drift | Motion can ignore layout intent |
| `SizeEffect` | Expanding or collapsing the occupied area | Siblings move during the animation |

Use `AlignEffect` when the destination is semantic, such as “top”, “center”, or “bottom”. Use `MoveEffect` when the destination is a precise offset. Mixing both is possible, but it is easy to create a transition that looks like a diagonal jump because alignment and translation are being applied at the same time.

## Practical checks

- Keep the parent bounded with `SizedBox`, `ConstrainedBox`, or a flexible panel.
- Test both directions. A target-driven animation should look intentional when it reverses.
- Do not use alignment to solve a sizing problem. If the panel must grow, pair it with a deliberate size transition.
- Check hit testing after the move. The child is still laid out in its new visual position, but surrounding widgets may remain unchanged.

`AlignEffect` is a small effect, but it gives layout motion a clearer meaning than a pile of offsets. Define the available space, choose semantic alignments, and let `target` handle the state change.
