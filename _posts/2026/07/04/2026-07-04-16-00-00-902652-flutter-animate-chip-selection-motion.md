---
layout: post
title: "flutter_animate Chip Selection Motion - clearer filters without rebuilding the whole row"
description: "Use flutter_animate with ChoiceChip and FilterChip to show selection changes clearly without replaying every filter option."
date: 2026-07-04
tags: [flutter_animate, animation, performance, state_management]
comments: true
share: true
---

![Flutter chip selection animation on mobile filter controls](https://images.unsplash.com/photo-1512941937669-90a1b58e7e9c?auto=format&fit=crop&w=1200&q=80)
The useful detail in this image is the small control surface: chip motion should clarify one changed choice, not pull attention away from the whole screen.

`flutter_animate` works well with `ChoiceChip` and `FilterChip` when the animation is tied to the selected value, not to the list build. The mistake I see most often is wrapping every chip in the same entrance animation. It looks fine with 4 filters, then becomes noisy with 12 filters and a search field above it.

The calmer pattern is simple: keep the chip row stable, animate only the chip whose selected state changes, and use a short scale or tint change instead of a slide. Chips are already compact. A 24 px movement reads like layout instability.

| Chip case | Better motion | Trap to avoid |
|---|---|---|
| Single category | small scale pulse | replaying all choices |
| Multiple filters | color and elevation fade | long stagger on every tap |
| Disabled option | no motion | animating unavailable choices |

Here is the shape I use for a single-select category row. The stable `ValueKey` keeps identity tied to the option, while `target` decides whether that chip should animate.

```dart
class CategoryChips extends StatefulWidget {
  const CategoryChips({super.key});

  @override
  State<CategoryChips> createState() => _CategoryChipsState();
}

class _CategoryChipsState extends State<CategoryChips> {
  final categories = const ['All', 'Unread', 'Pinned', 'Archived'];
  String selected = 'All';

  @override
  Widget build(BuildContext context) {
    return Wrap(
      spacing: 8,
      runSpacing: 8,
      children: [
        for (final category in categories)
          ChoiceChip(
            key: ValueKey(category),
            label: Text(category),
            selected: selected == category,
            onSelected: (_) => setState(() => selected = category),
          )
              .animate(target: selected == category ? 1 : 0)
              .scale(
                begin: const Offset(1, 1),
                end: const Offset(1.04, 1.04),
                duration: 140.ms,
                curve: Curves.easeOutCubic,
              )
              .then()
              .scale(
                end: const Offset(1, 1),
                duration: 90.ms,
                curve: Curves.easeOut,
              ),
      ],
    );
  }
}
```

For multi-select filters, I usually skip the second scale step and let Material's selected color do most of the work. The animation should only answer "did my tap register?" A short elevation or shadow fade is enough.

One edge case matters on real screens: chips often sit inside a `ListView` header or a pinned `SliverPersistentHeader`. If the parent rebuilds after every query change, the chips can replay even when selection did not change. Keep the selected filter state outside the search text state, or split the chip row into its own widget with stable inputs.

The practical rule is to animate state transitions, not the chip collection. If one chip changes, one chip should move. That keeps filter controls readable, especially when the same screen also contains loading rows, validation messages, or route transitions.
