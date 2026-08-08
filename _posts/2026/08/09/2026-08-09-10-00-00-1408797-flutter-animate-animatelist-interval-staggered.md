---
layout: post
title: "flutter_animate AnimateList.interval - Build Staggered Flutter Animations Without Manual Delays"
description: "Learn how flutter_animate AnimateList.interval staggers Flutter list animations, chooses a safe interval, and avoids common rebuild and key problems."
date: 2026-08-09
tags: [flutter_animate, animation, performance, ListView]
comments: true
share: true
---

![flutter_animate AnimateList interval staggered animation](/assets/images/flutter-animate-staggered-timing.png)

The cleanest way to stagger a group of Flutter widgets with `flutter_animate` is `AnimateList.interval`. It applies the same effect chain to every item and shifts each item's timeline by a fixed amount, so you do not have to calculate a different `delay` for every child.

## The problem with hand-written delays

Animating five cards individually usually starts like this: card one gets `delay: 0.ms`, card two gets `delay: 80.ms`, and so on. It works until the list changes. Insert one item and every delay below it becomes stale. A copied delay also makes it easy for a long list to outlive the parent animation, producing a slow, awkward entrance.

`AnimateList.interval` keeps the timing rule beside the list instead of scattering it through the item builder.

## A practical staggered list

This example fades and slides each card in. `interval` is the gap between the starts of neighboring items, not the total duration of each item.

```dart
final cards = <Widget>[
  const StatusCard(title: 'Build complete', icon: Icons.check),
  const StatusCard(title: 'Tests passing', icon: Icons.verified),
  const StatusCard(title: 'Ready to deploy', icon: Icons.rocket_launch),
];

AnimateList(
  interval: 90.ms,
  effects: [
    FadeEffect(
      duration: 320.ms,
      curve: Curves.easeOut,
    ),
    SlideEffect(
      begin: const Offset(0, 0.18),
      duration: 420.ms,
      curve: Curves.easeOutCubic,
    ),
  ],
  children: cards,
)
```

The first item begins immediately. The second begins about 90 ms later, and the third about 180 ms later. Because the effects themselves last longer than the interval, the cards overlap in motion. That overlap is what makes the list feel continuous rather than like three separate animations.

| Setting | What it controls | A useful starting point |
| --- | --- | --- |
| `interval` | Start-time gap between items | `60–120.ms` |
| `duration` | Length of each item's effect | `280–500.ms` |
| `begin` | Initial visual position | `Offset(0, 0.12)` |
| `curve` | Motion character | `Curves.easeOutCubic` |

## Choosing the interval

For a short dashboard, I usually start at `80.ms`. At `300.ms` the list feels serial: the user watches each card arrive. At `20.ms`, the stagger is technically present but visually reads as one large animation. The right value depends on item height and the amount of information on screen, so test on a real device rather than tuning only in a hot-reload preview.

There is also a useful constraint: if the list has `n` items, the final item's start is approximately `(n - 1) * interval`. A 20-row list with a 100 ms interval starts its last row almost two seconds after the first. For long lists, use a smaller interval, animate only the first visible group, or use a viewport-driven pattern instead.

## Two traps that look like animation bugs

`AnimateList` does not make unstable list identity safe. When the data can be reordered, give each item a stable key. Without one, Flutter may reuse an element for a different card and the animation appears to replay on the wrong row.

Also, do not rebuild the entire `AnimateList` for every keystroke or timer tick. A new animation tree can restart the entrance sequence. Keep changing state below the animated item when possible, or give the `Animate` widget a deliberate `key` when a replay is actually wanted.

For a single widget chain, `delay` is still the better tool. For sequencing different effects on one widget, use `ThenEffect`. `AnimateList.interval` is specifically the list-level rule: same effects, offset start times.

## Short checklist

- Use `AnimateList.interval` when every item shares the same effect chain.
- Keep the interval shorter than the effect duration for overlapping motion.
- Reduce the interval as the list gets longer.
- Add stable keys when items can be inserted, removed, or reordered.
- Check reduced-motion behavior before shipping a decorative entrance.

The API details are in the [official flutter_animate documentation](https://pub.dev/documentation/flutter_animate/latest/flutter_animate/). The important design choice is simple: define one motion recipe, then control rhythm with one interval.
