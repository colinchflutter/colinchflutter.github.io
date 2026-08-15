---
layout: post
title: "flutter_animate Effect Order - Combine SizeEffect and FadeEffect Without Layout Flicker"
description: "Learn how flutter_animate effect order changes Flutter layout behavior when combining SizeEffect and FadeEffect for smooth expandable status panels."
date: 2026-08-15
tags: [flutter_animate, animation, performance, Flutter]
comments: true
share: true
---

![flutter_animate SizeEffect and FadeEffect controlling a Flutter status panel](/assets/images/flutter-animate-curves-motion.png)

*The key difference is visible in the layout slot: opacity hides pixels, while size changes the space occupied by the widget.*

When a Flutter panel opens or closes, `flutter_animate` effect order decides whether the surrounding layout moves naturally or appears to flicker. `FadeEffect` changes visibility but keeps the original space. `SizeEffect` changes the layout slot itself. Combining them in the right order produces a panel that shrinks while its contents fade, without a separate `AnimationController`.

## The layout problem behind a simple fade

I first tried to close a filter panel with `.fadeOut()`. The panel became transparent, but the list below stayed in the same position until the widget was removed. That created an empty gap and made the close action feel unfinished.

Using only `.size()` fixed the gap, but the text and controls became visibly clipped as the height collapsed. The useful combination is a short fade inside a longer size transition:

| Effect | Changes | Typical duration | Layout result |
|---|---|---:|---|
| `fadeOut` | Child opacity | 140 ms | Space remains |
| `size` | Child dimensions | 260 ms | Neighbors move |
| `slideY` | Child paint position | 220 ms | Pixels move inside the slot |

The effects can overlap, but their responsibilities should remain separate. Fade makes content disappear; size removes the empty layout space.

## Build an expandable status panel

This example starts with the panel visible and plays the exit sequence when the widget is replaced with a keyed closing state. The `fadeOut` effect is placed before `size` so the controls become quiet before their layout slot reaches zero.

```dart
import 'package:flutter/material.dart';
import 'package:flutter_animate/flutter_animate.dart';

class FilterPanel extends StatelessWidget {
  const FilterPanel({super.key, required this.onClose});

  final VoidCallback onClose;

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        Row(
          children: [
            const Expanded(
              child: Text(
                'Filters',
                style: TextStyle(fontWeight: FontWeight.bold),
              ),
            ),
            IconButton(
              onPressed: onClose,
              icon: const Icon(Icons.close),
            ),
          ],
        ),
        const SizedBox(height: 12),
        const Wrap(
          spacing: 8,
          children: [
            Chip(label: Text('Popular')),
            Chip(label: Text('Nearby')),
            Chip(label: Text('Open now')),
          ],
        ),
      ],
    )
        .animate()
        .fadeIn(duration: 160.ms)
        .slideY(begin: -.04, end: 0, duration: 220.ms);
  }
}
```

For the closing state, keep the same visual content and reverse the effect order through an explicit controller, or let a parent transition remove the keyed child. A simple `AnimatedSwitcher` is often the cleanest boundary:

```dart
AnimatedSwitcher(
  duration: 280.ms,
  transitionBuilder: (child, animation) {
    return SizeTransition(
      sizeFactor: animation,
      axisAlignment: -1,
      child: FadeTransition(opacity: animation, child: child),
    );
  },
  child: showFilters
      ? FilterPanel(
          key: const ValueKey('filters'),
          onClose: onClose,
        )
      : const SizedBox(key: ValueKey('empty-filter-panel')),
)
```

Here Flutter owns the enter and exit animation, so the layout change is predictable. `flutter_animate` remains a good fit for the panel's internal entrance effects, such as the small title slide. Mixing both systems on the same property is where trouble begins.

## Avoid two animations fighting over height

Do not animate the panel's height with `SizeEffect` and `AnimatedSize` at the same time. Both widgets listen to a changing dimension, and the result can be a delayed collapse or a one-frame jump. Pick one owner for layout size:

- Use `AnimatedSize` when the child naturally changes its intrinsic height.
- Use `SizeEffect` when the animation belongs to a declarative `flutter_animate` chain.
- Use `FadeEffect` or `SlideEffect` for visual treatment inside the layout slot.

The same rule applies to `AnimatedSwitcher`. Let it own the replacement size, then use `flutter_animate` on descendants rather than wrapping the entire replacement in another size animation.

## Check the close interaction on real content

Short labels can hide layout bugs. Test the panel with two-line text, a large text setting, and a narrow phone width. A `Wrap` may take an extra row, which changes the measured height while the animation is running.

Also keep semantics in mind. A transparent widget may still be reachable by accessibility tools while `FadeEffect` is active. If the panel should no longer be actionable, update its state or remove it after the transition. Visual opacity alone is not a visibility contract.

The practical rule is simple: assign layout changes to one widget, visual changes to another, and keep their durations close enough that the user reads them as one action. That separation makes `flutter_animate` chains easier to tune and prevents a disappearing Flutter panel from leaving a mysterious hole behind.

