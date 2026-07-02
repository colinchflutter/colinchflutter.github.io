---
layout: post
title: "flutter_animate ExpansionTile - reveal motion for expandable rows"
description: "Learn how to add flutter_animate reveal motion to ExpansionTile content without breaking tile state or replaying every row."
date: 2026-07-03
tags: [flutter_animate, animation, ListView, Flutter, performance]
comments: true
share: true
---

![Flutter ExpansionTile reveal motion with flutter_animate](https://images.unsplash.com/photo-1515879218367-8466d910aaa4?w=800&q=80)

Use `flutter_animate` on the content inside `ExpansionTile`, not on the tile shell. The tile already owns height, icon rotation, tap handling, and semantics. Your job is to make the revealed content feel intentional without resetting every row in the list.

The problem shows up in settings screens and FAQ pages. You tap one row, the children appear, and someone wraps the entire `ExpansionTile` with `.animate().fadeIn()`. It looks fine in a tiny demo, then a real `ListView` rebuilds and half the visible rows replay their entrance animation. I have shipped that bug once. It makes a simple preferences page feel unstable.

Keep expansion state outside the animated child. A `Set<int>` is enough when the list is small and local.

```dart
class AnimatedFaqList extends StatefulWidget {
  const AnimatedFaqList({super.key});

  @override
  State<AnimatedFaqList> createState() => _AnimatedFaqListState();
}

class _AnimatedFaqListState extends State<AnimatedFaqList> {
  final _openIndexes = <int>{};

  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      itemCount: faqs.length,
      itemBuilder: (context, index) {
        final item = faqs[index];
        final isOpen = _openIndexes.contains(index);

        return ExpansionTile(
          key: PageStorageKey('faq-${item.id}'),
          title: Text(item.question),
          initiallyExpanded: isOpen,
          onExpansionChanged: (expanded) {
            setState(() {
              if (expanded) {
                _openIndexes.add(index);
              } else {
                _openIndexes.remove(index);
              }
            });
          },
          children: [
            Padding(
              padding: const EdgeInsets.fromLTRB(16, 0, 16, 16),
              child: Text(item.answer)
                  .animate(
                    key: ValueKey('answer-${item.id}-$isOpen'),
                    target: isOpen ? 1 : 0,
                  )
                  .fadeIn(duration: 180.ms, curve: Curves.easeOut)
                  .slideY(
                    begin: -0.08,
                    end: 0,
                    duration: 220.ms,
                    curve: Curves.easeOutCubic,
                  ),
            ),
          ],
        );
      },
    );
  }
}
```

The `PageStorageKey` preserves expansion behavior when the list scrolls. The `ValueKey` on `animate` is there for a different reason: it gives the revealed answer a clean animation identity when the open state changes. Without it, repeated open-close-open testing can reuse animation state in ways that feel random.

Do not animate height manually here. `ExpansionTile` already animates its vertical layout. Adding `SizeTransition`, `AnimatedSize`, or a custom `heightFactor` around the children usually creates double easing: the tile expands while the child also clips itself. The result is a slow, rubbery panel that clips text for a frame or two.

For dense admin screens, I use a shorter version:

```dart
child: Column(
  crossAxisAlignment: CrossAxisAlignment.start,
  children: [
    Text(item.answer),
    const SizedBox(height: 12),
    FilledButton(
      onPressed: () {},
      child: const Text('Apply'),
    ),
  ],
)
    .animate(target: isOpen ? 1 : 0)
    .fadeIn(duration: 140.ms)
    .moveY(begin: -6, end: 0, duration: 180.ms),
```

That small upward move is enough. Bigger offsets make the content look like it is sliding out from behind the header, which competes with the built-in expansion motion.

One edge case matters: lazy lists reuse rows. Never key animation state by the builder `index` if the list can be filtered or reordered. Use a stable item id. Index keys are fine only for static FAQ content that never changes order.

Short version: let `ExpansionTile` handle expansion, let `flutter_animate` handle content reveal, and keep stable keys on both layers. Animate opacity and a few pixels of motion. Avoid custom height animation unless you are replacing `ExpansionTile` entirely.
