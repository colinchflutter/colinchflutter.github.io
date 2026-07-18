---
layout: post
title: "flutter_animate AnimateList - Stagger List Items Without Replay Bugs"
description: "Learn how to use flutter_animate AnimateList for staggered Flutter list animations, stable keys, and dynamic updates that do not replay every item."
date: 2026-07-19
tags: [flutter_animate, animation, ListView, performance, Flutter]
comments: true
share: true
---

![A Flutter list UI with staggered animated cards](/assets/images/flutter-animate-animate-list-stable-keys.png)

`flutter_animate`'s `AnimateList` is a convenient way to give a group of Flutter widgets a staggered entrance animation. The part that needs care is not the fade or slide effect itself. It is deciding when the list is rebuilt. If the whole `AnimateList` is recreated after every state change, old rows can replay and make a simple refresh feel broken.

## The short version

Use `AnimateList` for a fixed batch of children, give data-driven rows a `ValueKey`, and keep the animated boundary close to the item that actually changed.

| Situation | Better choice |
| --- | --- |
| Initial batch of cards | `AnimateList` with an interval |
| A row updates in place | Keep its key and animate the row content |
| Items are inserted or removed | `AnimatedList` or a keyed transition wrapper |
| Every rebuild replays everything | Move `AnimateList` below the state boundary |

The table matters because `AnimateList` wraps each child in an `Animate` widget. It is excellent at coordinating a list of widgets, but it is not a diffing engine for arbitrary collection changes.

## A simple staggered list

This example animates a small set of notification cards once when the screen is built.

```dart
import 'package:flutter/material.dart';
import 'package:flutter_animate/flutter_animate.dart';

class NotificationList extends StatelessWidget {
  const NotificationList({super.key});

  @override
  Widget build(BuildContext context) {
    final notifications = [
      'Build finished',
      'New review comment',
      'Backup completed',
    ];

    final children = notifications.asMap().entries.map((entry) {
      return NotificationTile(
        key: ValueKey(notifications[entry.key]),
        label: entry.value,
      );
    }).toList();

    return Column(
      children: AnimateList<Widget>(
        interval: 90.ms,
        effects: [
          FadeEffect(duration: 260.ms, curve: Curves.easeOut),
          SlideEffect(
            begin: const Offset(0, .12),
            end: Offset.zero,
            duration: 260.ms,
            curve: Curves.easeOutCubic,
          ),
        ],
        children: children,
      ),
    );
  }
}
```

`interval: 90.ms` offsets the start time of each child. With three rows and a 260 ms effect, the last row starts 180 ms after the first one. That is enough to show order without making the list feel slow.

The same pattern works with the extension API:

```dart
Column(
  children: children.animate(interval: 90.ms).fadeIn().slideY(begin: .12),
)
```

I prefer the declarative constructor in shared components because the list boundary and the effect list are visible in one place. The extension reads nicely for a one-off screen.

## The rebuild trap

Consider a search screen where `setState` replaces `notifications` after every query. If `AnimateList` is inside that build method, every result creates a new group of `Animate` wrappers. Depending on the surrounding tree, rows that were already visible may fade in again. Users usually interpret that as flicker, not feedback.

For a changing collection, separate the stateful query area from the item transition:

```dart
class Results extends StatelessWidget {
  const Results({required this.items, super.key});

  final List<Result> items;

  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      itemCount: items.length,
      itemBuilder: (context, index) {
        final item = items[index];
        return ResultTile(
          key: ValueKey(item.id),
          item: item,
        );
      },
    );
  }
}
```

Here the key is not decoration. When filtering changes the index of an existing result, Flutter can still identify that result by `id`. Without a stable key, a stateful row may inherit another row's state, and any animation attached to that state becomes difficult to reason about.

If the requirement is specifically “animate only newly inserted rows,” use Flutter's `AnimatedList` for the collection operation and place a `flutter_animate` effect inside the row only when it is inserted. `AnimateList` is better suited to a known group of children entering together, such as an empty-state replacement or an initial dashboard load.

## Practical limits

Avoid a large interval for long lists. At 90 ms, item 20 would start 1.71 seconds after item 1. That is charming for six cards and irritating for a feed. For more than about eight visible items, animate the first viewport only, or use a scroll-triggered effect instead.

Also test the screen with reduced motion enabled. A short fade is usually harmless, but a long slide on every navigation event can become tiring. Keep the effect duration around 200–300 ms for ordinary cards, and make sure the list is usable before the animation completes.

The reliable mental model is simple: `AnimateList` coordinates a batch; keys preserve identity; `AnimatedList` owns collection changes. Keeping those responsibilities separate prevents most replay bugs.

### Key points

- Use `AnimateList` when several existing children enter as one visual batch.
- Set a stable `ValueKey` from the item's real ID, not its current index.
- Do not expect `AnimateList` to calculate insert and remove diffs.
- For long or frequently changing lists, prefer keyed rows and targeted transitions.
